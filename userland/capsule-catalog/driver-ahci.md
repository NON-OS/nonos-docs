# capsule_driver_ahci (full reference)

`capsule_driver_ahci` is the SATA storage driver in the NONOS tree: a userland capsule that owns one
AHCI host bus adapter (HBA), enumerates its ports, brings up the first usable SATA disk, and serves
sector read, write, and flush over IPC. It is a block-device backend. All partition, filesystem,
encryption, and cache policy stays above it in separate storage capsules; this capsule only moves
sectors and reports controller state. It reaches hardware exclusively through the
[hardware broker](../../subsystems/hardware-broker/README.md), never through kernel driver code.

It is spawned by the kernel as part of the storage driver fleet under service handle `driver.ahci0` on
service port 4216 with a reply port on 4217, and its manifest capability mask is `0xf8019`
(`userland/capsule_driver_ahci/Capsule.mk:16`). The source is `userland/capsule_driver_ahci/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [Architecture and bring-up](#architecture-and-bring-up)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The capsule is `no_std`/`no_main`. `_start` initialises the heap, runs the one-shot setup sequence, and
then hands the resulting `Driver` to the server loop; if setup fails it exits with a code derived from
the error (`userland/capsule_driver_ahci/src/main.rs:38`). The two halves are clean: `setup::run`
performs every privileged bring-up step once and returns a `Driver` that owns the broker grants and the
mapped register window, and `server::run` is an endless request loop that never touches the broker again
except to poll and acknowledge the controller interrupt (`src/setup/sequence.rs:26`,
`src/server/runner.rs:29`).

Setup discovers the AHCI PCI function, claims it, enables bus mastering, maps the ABAR register window,
binds the controller interrupt, enables AHCI mode, reads the controller-global registers, scans the
implemented ports, and brings up the first present SATA port with its command-list, received-FIS,
command-table, PRDT, and data DMA regions (`src/setup/sequence.rs:27`). Once a port is up, the driver
issues ATA `IDENTIFY` to record the disk's sector count and is ready to serve block I/O. The I/O path is
poll-driven: it programs command slot 0, writes `PxCI`, and spins on the completion registers rather than
waiting on the interrupt, so the driver works on platforms with no legacy interrupt routing
(`src/engine/issue.rs:22`, `src/discover.rs:60`).

## Identity

Everything the kernel and the service registry need to name and reach the driver comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-ahci` | `Capsule.mk:6` |
| Service handle | `driver.ahci0` | `Capsule.mk:7`, `src/hardware/ahci_capsule/spawn.rs:32` |
| Namespace | `systems.nonos.driver.ahci0` | `Capsule.mk:12` |
| Service endpoint | `service:4216:driver.ahci0` | `Capsule.mk:13`, `spawn.rs:33` |
| Reply endpoint | `reply:4217:endpoint.4294967311` | `Capsule.mk:14`, `spawn.rs:34` |
| Reply inbox (kernel) | `endpoint.4294967311` (`0x1_0000_000f`) | `src/hardware/ahci_capsule/client/transport.rs:25`, `src/protocol/endpoint.rs:18` |
| Capability mask | `0xf8019` | `Capsule.mk:16` |
| Binary name | `driver_ahci` | `Capsule.mk:10` |
| Kernel mirror | `src/hardware/ahci_capsule` | `src/hardware/ahci_capsule/mod.rs` |

The reply endpoint name `endpoint.4294967311` is the decimal spelling of the reply-inbox id the capsule
sends every response to. That id is `KERNEL_REPLY_ENDPOINT = 0x1_0000_000f` in the capsule
(`src/protocol/endpoint.rs:18`) and `REPLY_INBOX = "endpoint.4294967311"` on the kernel side
(`src/hardware/ahci_capsule/client/transport.rs:25`); `0x1_0000_000f` is 4294967311, so the two agree.

The mask `0xf8019` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x00001  CoreExec     bit()      1     types.rs:56
  0x00008  IPC          bit()      8     types.rs:59
  0x00010  Memory       bit()     16     types.rs:60
  0x08000  DeviceEnum   bit()  32768     types.rs:71
  0x10000  Driver       bit()  65536     types.rs:72
  0x20000  Mmio         bit() 131072     types.rs:73
  0x40000  Irq          bit() 262144     types.rs:74
  0x80000  Dma          bit() 524288     types.rs:75
  -------
  0xf8019  = 1 + 8 + 16 + 32768 + 65536 + 131072 + 262144 + 524288
```

Those are exactly the capabilities a bus-mastering MMIO device driver needs and nothing more.
`DeviceEnum` is enumerate-only, `Driver` allows claim and release of one device, `Mmio` maps a device's
register window, `Irq` binds its interrupt, and `Dma` allocates device-visible buffers; the comment on
the capability enum spells out that split (`src/capabilities/types.rs:35`). There is no `Pio` bit
(0x100000), no `FileSystem` bit (64), no `Network` bit (4), and no `Admin` (512) or `Debug` (256). The
driver cannot touch an I/O port, read a file, open a socket, or reach raw kernel memory.

Two identity notes worth recording, because they are places where the code and its neighbours can drift:

- The `Capsule.mk` comment on line 15 lists `IPC|Memory|Driver|DeviceEnum|Mmio|Irq|Dma = 0xf8019`, but
  that seven-name set sums to `0xf8018`. The extra `0x1` in the real mask is `CoreExec`, which every
  runnable capsule needs and which the comment simply omits. The numeric value `0xf8019` on line 16 is
  the authority that is signed into the manifest, and it is the one decomposed above.
- The kernel-side spawn record requests only the seven hardware and IPC bits and not `CoreExec`
  explicitly (`src/hardware/ahci_capsule/spawn.rs:51`). The signed manifest mask `0xf8019`
  (`Capsule.mk:16`) is the ceiling the trust anchor enforces; the spawn `requested_caps` is a request
  bounded by that manifest, and `CoreExec` is granted to the capsule as a matter of being an executable
  process.

## Operation reference

The service accepts one request at a time on its endpoint, decodes the 20-byte header, and dispatches on
the operation id (`src/server/runner.rs:39`). Every reply reuses the same header shape and begins with a
4-byte little-endian status word; a non-negative status is success and a negative status is one of the
errno values below (`src/protocol/errno.rs:17`). The opcodes are defined in `src/protocol/ops.rs`.

Errno values (`src/protocol/errno.rs:17`):

```
  E_OK       0     success
  E_IO      -5     command issued but the controller reported an error or timed out
  E_NODEV  -19     no block port was brought up at setup
  E_INVAL  -22     bad opcode, or a fixed-size op carried an unexpected payload
  E_NXIO    -6     requested LBA range runs past the disk capacity
  E_MSGSIZE -90    request length did not match the op's declared layout
```

| Op | Opcode | Request payload | Reply payload | Errors | Handler |
|---|---|---|---|---|---|
| `OP_HEALTHCHECK` | 1 | none | status word only | `E_INVAL` if payload present | `ops.rs:17`, `handlers/health.rs:20` |
| `OP_CONTROLLER_INFO` | 2 | none | status + 24-byte register record | `E_INVAL` if payload present | `ops.rs:18`, `handlers/controller_info.rs:25` |
| `OP_PORT_LIST` | 3 | none | status + 4-byte count + 36-byte port entries | `E_INVAL` if payload present | `ops.rs:19`, `handlers/port_list.rs:26` |
| `OP_CAPACITY` | 4 | none | status + 8-byte sector count | `E_NODEV` if no port | `ops.rs:20`, `handlers/capacity.rs:26` |
| `OP_READ_BLOCKS` | 5 | 12-byte `lba, count` | status + `count * 512` bytes | `E_NODEV`, `E_MSGSIZE`, `E_INVAL`, `E_NXIO`, `E_IO` | `ops.rs:21`, `handlers/read.rs:28` |
| `OP_WRITE_BLOCKS` | 6 | 12-byte header + `count * 512` bytes | status word only | `E_NODEV`, `E_INVAL`, `E_NXIO`, `E_MSGSIZE`, `E_IO` | `ops.rs:22`, `handlers/write.rs:23` |
| `OP_FLUSH` | 7 | none | status word only | `E_NODEV`, `E_IO` | `ops.rs:23`, `handlers/flush.rs:22` |

Request and reply header (20 bytes, `src/protocol/header.rs:17`, `src/protocol/decode.rs:19`,
`src/protocol/encode.rs:19`):

```
  u32 magic      = 0x4e41_4843  "NAHC"   header.rs:17
  u16 version    = 1                     header.rs:18
  u16 op                                 decode.rs:30
  u16 flags                              decode.rs:31
  u16 reserved                           encode.rs:24
  u32 request_id                         decode.rs:32
  u32 payload_len                        decode.rs:33
```

A request whose first four bytes are not `NAHC` or whose version is not 1 is dropped with `E_INVAL` and
no port is touched (`src/protocol/decode.rs:23`, `src/server/error.rs:29`).

The healthcheck, controller-info, and port-list ops are fixed-size and reject any request that carries a
payload before they run (`src/server/runner.rs:57`). `OP_CONTROLLER_INFO` re-reads the HBA global
registers live at request time and returns `CAP`, `GHC`, `PI`, `VS` (version), `CAP2`, and the port
count (`handlers/controller_info.rs:26`). `OP_PORT_LIST` returns the snapshot taken at setup: for each
implemented port a 36-byte record of index, implemented flag, present flag, kind, and the six status
registers `PxSSTS`, `PxSIG`, `PxIS`, `PxCMD`, `PxTFD`, `PxSERR`, `PxSACT`, `PxCI`
(`handlers/port_list.rs:34`).

Read and write are bounded and range-checked. `rw_parse::parse` reads the 8-byte LBA and 4-byte sector
count, rejects a zero or over-`MAX_SECTORS` (64) count with `E_INVAL`, and rejects any `lba + count` that
overflows or exceeds the disk's identified capacity with `E_NXIO`
(`src/server/handlers/rw_parse.rs:20`, `src/constants/ata.rs:30`). A read copies the completed sector
bytes out of the DMA data region into the reply (`handlers/read.rs:48`); a write copies the request body
into the data region first, then issues the command and replies with a status word only
(`handlers/write.rs:37`). Write additionally checks that the body length is exactly the 12-byte header
plus `count * 512` before it copies (`handlers/write.rs:34`). `OP_FLUSH` issues ATA `FLUSH CACHE EXT`
with no data transfer (`handlers/flush.rs:28`, `src/engine/flush.rs:27`).

## Architecture and bring-up

The whole privileged path runs once in `setup::run` and unwinds prior broker grants on any failure
(`src/setup/sequence.rs:26`). The steps, in order:

1. Discover. `find_ahci` asks the broker for the block-class device list and returns the first record
   that is a PCI storage controller with subclass SATA and prog-IF AHCI, with BAR5 present as MMIO of at
   least 4 KiB (`src/discover.rs:34`, `:52`). The IRQ line is deliberately not part of the match: a
   controller that reports `irq_line = 0xff` (common on APIC/MSI laptop platforms with no legacy PIC
   routing) is still a valid candidate, because the I/O path polls completion and never waits on the
   interrupt. The comment records that requiring a routed line here previously rejected working SATA
   controllers on real hardware (`src/discover.rs:60`).
2. Claim. `mk_device_claim` binds the controller to this process and returns a claim epoch; every later
   broker call carries that epoch (`src/setup/claim.rs:21`).
3. Bus master. `enable_bus_master` writes the PCI command register to set the bus-master bit, which the
   controller needs to drive DMA (`src/setup/pci.rs:21`). Failure here releases the device before
   returning (`src/setup/sequence.rs:29`).
4. Map ABAR. `mmio::map` maps BAR5, the AHCI register window, into the capsule's address space and
   returns a user VA and a grant id; failure releases the device (`src/setup/mmio.rs:22`,
   `src/constants/pci.rs:18`). A thin `Regs` wrapper does volatile 32-bit reads and writes at
   `base + offset` (`src/regs/mmio.rs:29`).
5. Bind IRQ. `irq::bind` binds the controller's interrupt line and returns a grant id; failure unmaps
   the ABAR and releases the device (`src/setup/irq.rs:22`). The interrupt is only polled and acked at
   runtime; it is not required for command completion.
6. Enable AHCI mode. `enable_ahci` sets `GHC.AE` (bit 31) and clears the global interrupt status by
   writing all-ones to `HBA_IS` (`src/controller/enable.rs:20`, `src/constants/regs.rs:24`).
7. Read controller info and scan ports. `ControllerInfo::read` derives the port count from `CAP` bits
   0..4 plus one and records `CAP`, `GHC`, `PI`, `VS`, `CAP2` (`src/controller/info.rs:31`).
   `scan_ports` walks each bit set in `PI`, computes the port register base as `0x100 + index * 0x80`,
   reads `PxSSTS` and `PxSIG`, marks a port present only when `DET == 3` and `IPM` is 1 or 6, and
   classifies the port from its signature (`src/controller/scan_ports.rs:26`, `:65`,
   `src/controller/signature.rs:20`).
8. Bring up the block port. `block_port::bring_up` picks the first present SATA-signature port and calls
   `init_port` (`src/setup/block_port.rs:22`).

### Command list, FIS, and command table

`init_port` allocates four DMA regions through the broker: the command list base (`clb`), the command
table base (`ctba`), the received-FIS base (`fb`), each a 4 KiB struct region, and a data buffer of
`MAX_SECTORS * 512` (32 KiB) (`src/engine/init.rs:24`, `src/constants/ata.rs:30`). Each region carries a
device-visible physical address and a user VA; `DmaRegion` unmaps itself on drop (`src/engine/region.rs:46`).

The three hardware structures are fixed C layouts with compile-time size assertions:

```
  CmdHeader   32 bytes   flags, prdtl, prdbc, ctba_low/high   engine/cmd_header.rs:19, :29
  CmdTable   144 bytes   cfis[64], acmd[16], rsv[48], prdt[1]  engine/cmd_table.rs:21, :28
  FisH2D      20 bytes   Register Host-to-Device FIS           engine/fis.rs:19, :39
  PrdtEntry   16 bytes   dba_low/high, dbc                     engine/prdt.rs:19, :26
```

After allocation, `program` writes command header 0 to point at the command table's physical address,
programs `PxCLB`/`PxCLBU` and `PxFB`/`PxFBU` with the command-list and FIS physical addresses, clears
`PxIS` and `PxSERR`, sets the port interrupt-enable default, disables aggressive link power management in
`PxSCTL`, and spins the port up with `PxCMD.POD | PxCMD.SUD` (`src/engine/program.rs:24`,
`src/constants/regs.rs:43`). `stop` then clears `PxCMD.ST` and `PxCMD.FRE` and waits for the command and
FIS-receive engines to go idle, and `start` waits for `CR` to clear before setting `FRE | ST`
(`src/engine/stop.rs:21`, `src/engine/start.rs:21`). Both spins are bounded by `COMPLETION_POLL_LIMIT`
(`src/constants/ata.rs:33`).

There is no controller-wide `GHC.HR` HBA reset in this slice. The bring-up asserts `GHC.AE` and then does
a per-port engine stop/start cycle instead: `stop` idles the port before its command list and FIS base are
reprogrammed, and `start` re-enables the engines afterward. That per-port cycle, together with the
discovery change that stopped rejecting controllers whose `irq_line` is `0xff`, is what makes the driver
come up on real x86_64 SATA controllers as well as on QEMU's `ich9-ahci` (`src/engine/stop.rs:21`,
`src/engine/start.rs:21`, `src/discover.rs:60`). A full device reset is named as a future target in the
capsule README, not something this slice performs (`README.md:124`).

### The DMA and command path

A command is always issued in slot 0. `build_slot0` zeroes the command table, writes a Register H2D FIS
carrying the ATA command byte, the 48-bit LBA, the sector count, and (for anything but `IDENTIFY`) the
LBA-mode device bit, writes one PRDT entry pointing at the data buffer's physical address with a
byte-count-minus-one length, and writes command header 0 with the FIS length in dwords and the write flag
when the transfer is outbound (`src/engine/build.rs:26`, `src/engine/prdt_write.rs:19`). `issue_slot0`
then clears `PxIS`, waits for the task file to leave `BSY|DRQ`, writes 1 to `PxCI` to issue the command,
and polls until `PxCI` bit 0 clears (success), or an error appears in `PxIS` or `PxTFD.ERR` (returns
`CommandFailed`), or the poll limit is hit (returns `Timeout`) (`src/engine/issue.rs:22`). On any error
the caller runs `recover`, which clears `PxSERR` and `PxIS` and does a stop/start cycle before the error
propagates (`src/engine/recover.rs:20`).

`identify` issues ATA `IDENTIFY` (0xEC) into the data buffer and reads the 48-bit sector count from words
100..103, falling back to the 28-bit count in words 60..61 when the extended count is zero; the result is
stored as the port's capacity in sectors (`src/engine/identify.rs:22`). `transfer` issues `READ DMA EXT`
(0x25) or `WRITE DMA EXT` (0x35) for the requested LBA and count (`src/engine/transfer.rs:22`,
`src/constants/ata.rs:17`). `flush` issues `FLUSH CACHE EXT` (0xEA) with no PRDT (`src/engine/flush.rs:27`).

### IRQ handling

The server loop polls the controller interrupt each iteration through `mk_irq_poll` on the IRQ grant id;
when the sequence number advances it acknowledges with `mk_irq_ack` and records the new sequence
(`src/server/runner.rs:71`). This keeps the interrupt drained but does not gate command completion, which
is decided entirely by polling `PxCI` and the status registers in `issue_slot0`. That is why a controller
with no routed legacy line still works.

## Protocol and IPC

The driver is a server. It never calls another userland service; every outbound call it makes is a broker
syscall, and every inbound call is a block request on its endpoint.

Inbound, service `driver.ahci0` on `service:4216:driver.ahci0`: the loop blocks on `mk_ipc_recv`,
decodes the `NAHC` header, dispatches one of the seven ops above, and sends the reply to
`KERNEL_REPLY_ENDPOINT` with `mk_ipc_send` (`src/server/runner.rs:39`, `src/server/error.rs:26`).

Outbound broker syscalls, each through `nonos_libc` and each documented in the hardware broker subsystem:

```
  mk_device_list       enumerate block-class devices        setup/../discover.rs:36    (broker device list)
  mk_device_claim      claim the AHCI function               setup/claim.rs:22          claim.md
  mk_pci_config_write  set the PCI bus-master bit            setup/pci.rs:22            claim.md
  mk_mmio_map          map BAR5 / ABAR into the capsule VA   setup/mmio.rs:24           mmio.md
  mk_mmio_unmap        unmap ABAR on failure or teardown     setup/mmio.rs:26, handles/broker_handles_drop.rs:24   mmio.md
  mk_irq_bind          bind the controller interrupt         setup/irq.rs:24            irq.md
  mk_irq_poll          poll for interrupt events             server/runner.rs:73        irq.md
  mk_irq_ack           acknowledge a delivered interrupt     server/runner.rs:75        irq.md
  mk_irq_unbind        release the interrupt on teardown     handles/broker_handles_drop.rs:23   irq.md
  mk_dma_map           allocate command/FIS/table/data DMA   engine/region.rs:30        dma.md
  mk_dma_unmap         free a DMA region on drop             engine/region.rs:48        dma.md
  mk_device_release    release the device claim              setup/*, handles/broker_handles_drop.rs:25   claim.md
```

The [claim](../../subsystems/hardware-broker/claim.md), [mmio](../../subsystems/hardware-broker/mmio.md),
[dma](../../subsystems/hardware-broker/dma.md), and [irq](../../subsystems/hardware-broker/irq.md) broker
pages describe how each grant is validated, bounded to the claim epoch, and revoked. All of the driver's
broker grants are held in two owners that free them on drop: `BrokerHandles` releases the IRQ bind, the
ABAR mapping, and the device claim in that order (`src/handles/broker_handles_drop.rs:22`), and each
`DmaRegion` unmaps itself (`src/engine/region.rs:46`).

## Security analysis

The driver holds real hardware authority, and that authority is the point of the design: the block
policy that used to live in the kernel is pushed into a signed, sandboxed, single-device userland capsule
that can only reach the SATA controller through broker-mediated grants. Its mask `0xf8019` grants
`CoreExec`, `IPC`, `Memory`, `DeviceEnum`, `Driver`, `Mmio`, `Irq`, and `Dma` and nothing else
(`Capsule.mk:16`). There is no `Pio` bit, so the driver cannot fall back to raw port I/O; no `FileSystem`
bit, so it cannot read or write a file; no `Network` bit; and no `Admin` or `Debug`.

What the broker grants and bounds:

- The device claim binds exactly one AHCI function to this process and issues an epoch that every later
  MMIO, DMA, and IRQ call must carry (`src/setup/claim.rs:22`). A capsule cannot act on a device it did
  not claim, and the kernel revokes the claim and every grant behind it on exit.
- `mk_mmio_map` maps only BAR5, the ABAR window, and only for this claim; the driver never sees any other
  register space (`src/setup/mmio.rs:24`). Writes go through a volatile 32-bit wrapper at offsets inside
  that window (`src/regs/mmio.rs:29`).
- `mk_dma_map` returns device-visible physical addresses that the driver programs into the command list,
  FIS base, command table, PRDT, and data buffer; the driver never invents a physical address, it only
  uses the ones the broker handed it (`src/engine/region.rs:30`, `src/engine/program.rs:36`).
- The bus-master bit is set through `mk_pci_config_write` against the claimed function, not by poking
  config space directly (`src/setup/pci.rs:22`).

The no-IOMMU caveat is the honest limit. A bus-mastering SATA controller programmed with physical DMA
addresses can, at the silicon level, read or write any physical address it is told to. This driver only
ever programs addresses the broker returned from `mk_dma_map`, and range-checks every client LBA against
the identified disk capacity before issuing a command (`src/server/handlers/rw_parse.rs:29`). But on a
platform where the kernel has not engaged an IOMMU, the hardware itself does not enforce that the
controller stays inside those DMA regions; the enforcement is the driver's correctness plus the broker's
grant discipline, not a hardware boundary. That is the standard caveat for any bus-mastering device
driver on NONOS and is called out in the broker DMA page.

Isolation from other capsules is the kernel's, not the driver's: it is a CPL 3 user binary, verified and
enrolled at spawn like every other capsule, that speaks only IPC and broker syscalls. A bug in header
parsing, range checking, or command construction cannot escalate past the eight capabilities in the mask,
because the driver never held more than the right to claim one device and speak to it. Requests are
strictly validated before any hardware is touched: a bad magic or version is dropped, a fixed-size op
with a payload is rejected, a read or write with the wrong length gets `E_MSGSIZE`, an out-of-range LBA
gets `E_NXIO`, and a request that arrives before any port was brought up gets `E_NODEV`
(`src/server/runner.rs:57`, `src/server/handlers/rw_parse.rs:20`, `src/server/handlers/read.rs:32`).

## How to contribute

The source lives at `userland/capsule_driver_ahci/`. The tree is one unit per file: discovery is
`src/discover.rs`, the one-shot bring-up is under `src/setup/`, the controller-global reads and port scan
are under `src/controller/`, the command engine (DMA regions, FIS, command list and table, PRDT, issue,
identify, transfer, flush, recover) is under `src/engine/`, the wire format is under `src/protocol/`, the
request loop and per-op handlers are under `src/server/`, the broker-grant owners are under
`src/handles/`, and the register and ATA constants are under `src/constants/`.

To add or change an operation:

1. Add the opcode to `src/protocol/ops.rs` and any fixed sizes to `src/protocol/limits.rs`.
2. Write the handler as one file under `src/server/handlers/`, exposing a `pub fn handle(...)` that
   builds the reply with `encode_response_header` and `write_status` and sends it with `mk_ipc_send`, or
   returns an errno through `reply_with_status` (`src/server/error.rs:23`). Re-export it from
   `src/server/handlers/mod.rs`.
3. Wire it into the match in `src/server/runner.rs:56`, adding a payload-length guard if it is a
   fixed-size op.
4. If it needs a new ATA command, add the command byte to `src/constants/ata.rs` and the engine step
   under `src/engine/`, keeping the register offsets in `src/constants/regs.rs`.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:156`,
included through `userland/capsule_driver_ahci/Capsule.mk:18`):

```
  make nonos-mk-driver-ahci             build the capsule ELF
  make nonos-mk-driver-ahci-sign        produce the id cert, manifest, and attestation trailer
  make nonos-mk-driver-ahci-verify      verify the signed artifacts against the trust anchor
  make nonos-mk-check-driver-ahci-keys  check the per-capsule signing keys exist
```

For a running kernel that embeds and spawns the driver, `make nonos-mk-driver-ahci-prod` builds the
kernel image with the `microkernel-driver-ahci` feature so the storage fleet spawns `driver_ahci` at boot
(`Makefile:1005`). The README documents the same build and static-gate entry points (`README.md:152`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every fallible path returns an `AhciError` or an errno, and the release profile
is `panic = "abort"`, `Cargo.toml:26`); modular files, one unit per file, with `mod.rs` used only for
re-exports; and the AGPL header at the top of every source file, matching the header on every existing
module. The architecture gate additionally forbids importing `crate::drivers`, `crate::hardware`,
`crate::memory`, or `crate::paging`, and forbids inline PIO or DMA, so the only path to hardware stays
the broker (`README.md:154`).

## Debugging

The first thing to confirm is that the capsule ran. The storage fleet boots the driver under the tag
`DRIVER-AHCI` (`src/userspace/init/spawn_plan/drivers_storage.rs:27`), and a live driver prints a boot
line naming it; the sibling drivers page records `[DRIVER-AHCI]` as that marker
(`docs/userland/drivers.md:269`). If the ELF fails to load, the spawn path emits the debug tag
`[DRIVER-AHCI] load_elf_executable error:` (`src/hardware/ahci_capsule/spawn.rs:58`).

Setup failures are hard barriers and each maps to a distinct exit code, so a driver that never comes up
tells you where it stopped (`src/error/types.rs:27`):

```
  2  DeviceNotFound      no PCI AHCI SATA function matched discovery      discover.rs:34
  3  BrokerCallFailed    a claim/mmio/irq/dma/pci broker call returned <0  setup/*, engine/region.rs:31
  4  CommandFailed       IDENTIFY reported PxIS/PxTFD error at bring-up    engine/issue.rs:38
  5  Timeout             a command spun past COMPLETION_POLL_LIMIT         engine/issue.rs:45
```

Failure modes and where to look:

- No block port. If discovery found the controller but no present SATA-signature port came up,
  `driver.block` is `None` and every `OP_CAPACITY`, `OP_READ_BLOCKS`, `OP_WRITE_BLOCKS`, and `OP_FLUSH`
  returns `E_NODEV` (-19) while `OP_CONTROLLER_INFO` and `OP_PORT_LIST` still answer
  (`src/setup/block_port.rs:28`, `src/server/handlers/capacity.rs:27`). `OP_PORT_LIST` is the probe:
  check each entry's `present` flag, `PxSSTS` (`DET`/`IPM`), and `PxSIG` to see whether a disk is
  actually attached and what signature it reports (`src/controller/scan_ports.rs:65`,
  `src/controller/signature.rs:20`).
- Command timeout or error. A read, write, or flush that returns `E_IO` (-5) means `issue_slot0` saw an
  error in `PxIS`/`PxTFD` or spun past `COMPLETION_POLL_LIMIT`, after which `recover` cleared `PxSERR`
  and `PxIS` and cycled the port (`src/engine/issue.rs:35`, `src/engine/recover.rs:20`). Reissuing the
  same request after a recover is safe; a persistent `E_IO` points at the controller or the disk, not the
  wire format.
- Range or size rejects. `E_NXIO` (-6) is an LBA past the identified capacity, `E_INVAL` (-22) is a zero
  or over-64 sector count (or a bad opcode), and `E_MSGSIZE` (-90) is a read/write whose declared length
  did not match `12 + count * 512`. These are decided before any hardware access, so they are pure
  client-side protocol errors (`src/server/handlers/rw_parse.rs:26`, `src/server/handlers/write.rs:34`).
- A malformed request that is silently ignored. A frame whose magic is not `NAHC` or whose version is not
  1 is dropped inside `decode_request` and answered with `E_INVAL` via the decode-failed path, so a
  client that gets no useful reply should check that it is sending the 20-byte `NAHC` v1 header
  (`src/protocol/decode.rs:23`, `src/server/error.rs:29`).

## Source map

```
  src/main.rs                        _start -> setup::run -> server::run
  src/discover.rs                    find_ahci: PCI storage/SATA/AHCI match, irq_line 0xff tolerated
  src/setup/sequence.rs              the one-shot bring-up (claim, bus-master, mmio, irq, enable, scan, port)
  src/setup/{claim,pci,mmio,irq,block_port}.rs   the individual broker bring-up steps
  src/controller/{enable,info,scan_ports,signature,port_info}.rs   AHCI enable, global regs, port scan
  src/engine/init.rs                 init_port: allocate DMA, stop/program/start, identify
  src/engine/{program,stop,start,recover}.rs   port register programming and the stop/start cycle
  src/engine/{build,cmd_header,cmd_table,fis,prdt,prdt_write}.rs   command list, FIS, table, PRDT layout
  src/engine/{issue,identify,transfer,flush,region}.rs   issue slot 0, identify/read/write/flush, DMA region
  src/protocol/{header,decode,encode,ops,errno,limits,endpoint}.rs   the NAHC wire format
  src/server/runner.rs               the request loop, dispatch, and irq poll/ack
  src/server/handlers/               one file per op (health, controller_info, port_list, capacity, read, write, flush)
  src/server/error.rs                reply_with_status / reply_decode_failed
  src/handles/                       BrokerHandles: device, mmio, irq grants freed on drop
  src/constants/{regs,ata,port,pci}.rs   HBA/port register offsets, ATA commands, signatures
  Capsule.mk                         slug, handle, ports, capability mask
  src/hardware/ahci_capsule/         the kernel-side embed, verified spawn, and client
  src/userspace/init/spawn_plan/drivers_storage.rs   the storage-fleet spawn entry
  docs/subsystems/hardware-broker/   the claim, mmio, dma, and irq grant contracts
```

Every reference above is verified against those trees.
</content>
</invoke>

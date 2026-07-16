# capsule_driver_nvme (full reference)

`capsule_driver_nvme` is the NONOS NVMe storage-controller driver: a ring-3 capsule that drives a real
PCIe NVMe SSD and serves it to the rest of the system as a block device. It does not run in the kernel
and it does not touch the device through any privileged kernel path. It reaches its controller only
through the [hardware broker](../../subsystems/hardware-broker/README.md): it claims the PCI function,
maps BAR0, enables bus mastering, binds MSI-X, and allocates DMA, all as brokered grants scoped to a
claim epoch. Everything above those grants (the admin queue, the IO queue pair, Identify, SMART, and the
read/write/flush command path) is ordinary userland code inside the capsule.

The kernel spawns it under service handle `driver.nvme0` on service port 4220 with a reply inbox on
port 4221, and its capability mask is `0xF8019` (`userland/capsule_driver_nvme/Capsule.mk:16`). The
source is `userland/capsule_driver_nvme/`.

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

The capsule is `no_std`/`no_main`. `_start` initialises the heap, runs `setup::run`, and then hands the
built `Driver` to the request server, which loops forever (`userland/capsule_driver_nvme/src/main.rs:40`).
`setup::run` is the whole bring-up: it finds the NVMe function, claims it, enables bus mastering, maps
BAR0, binds MSI-X, resets and enables the controller, runs Identify Controller and Identify Namespace,
snapshots the SMART/health log, and creates one IO submission/completion queue pair
(`src/setup/sequence.rs:30`). If any of that fails the process exits with a distinct code and never
serves a request (`src/main.rs:45`, `src/error/types.rs:30`).

Once setup succeeds the capsule is a block-device backend. Clients speak a small binary protocol over IPC
(`src/protocol/`) to ask for controller info, controller and namespace identity, a SMART/health snapshot,
the namespace capacity, and to read, write, or flush 512-byte sectors on namespace 1
(`src/server/runner.rs:54`). The read and write paths move sector bytes through a brokered DMA buffer and
an NVM read/write command on the IO queue (`src/nvm/transfer.rs:25`). The capsule does not parse
partitions, mount filesystems, or cache payloads; it is the mechanism a higher-level storage service is
built on, not the policy.

## Identity

Everything the kernel and the service registry need to name and reach the driver comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-nvme` | `Capsule.mk:6` |
| Service handle | `driver.nvme0` | `Capsule.mk:7`, `src/hardware/nvme_capsule/spawn.rs:32` |
| Namespace | `systems.nonos.driver.nvme0` | `Capsule.mk:12` |
| Service endpoint | `service:4220:driver.nvme0` | `Capsule.mk:13`, `spawn.rs:33` |
| Reply endpoint | `reply:4221:endpoint.4294967313` | `Capsule.mk:14`, `spawn.rs:34` |
| Capability mask | `0xF8019` | `Capsule.mk:16` |
| Binary name | `driver_nvme` | `Capsule.mk:10`, `Cargo.toml:19` |
| Kernel mirror | `src/hardware/nvme_capsule` | `src/hardware/nvme_capsule/spawn.rs` |

The reply endpoint number `4294967313` is `0x1_0000_0011`, which is the kernel reply-inbox constant the
capsule sends every reply to (`src/protocol/endpoint.rs:18`, `src/server/error.rs:26`).

The mask `0xF8019` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x00008  IPC          bit()      8   types.rs:59
  0x00010  Memory       bit()     16   types.rs:60
  0x08000  DeviceEnum   bit()  32768   types.rs:71
  0x10000  Driver       bit()  65536   types.rs:72
  0x20000  Mmio         bit() 131072   types.rs:73
  0x40000  Irq          bit() 262144   types.rs:74
  0x80000  Dma          bit() 524288   types.rs:75
  -------
  0xF8019  = 8 + 16 + 32768 + 65536 + 131072 + 262144 + 524288
```

The kernel spawn path requests exactly those seven capabilities and no others
(`src/hardware/nvme_capsule/spawn.rs:51`), which matches the comment and value in the manifest
(`Capsule.mk:15`). Unlike an application capsule, this driver holds the hardware-broker authority bits:
`DeviceEnum` (enumerate devices), `Driver` (claim and release a device), `Mmio` (map device registers),
`Irq` (bind a device interrupt), and `Dma` (map DMA), the exact set the broker checks before it will hand
out any grant (`src/capabilities/types.rs:34`). It has no `Network` bit (4), no `FileSystem` bit (64), and
no graphics or raw-physmem authority. `IPC` and `Memory` are the only bits it shares with an ordinary app.

## Operation reference

The server decodes a 20-byte request header (`MAGIC = 0x4e4e_564d` "NNVM", version 1) and dispatches on
the 16-bit opcode (`src/protocol/header.rs:17`, `src/server/runner.rs:54`, `src/protocol/decode.rs:19`).
The five zero-payload query ops are rejected with `E_INVAL` if the client sends any payload
(`src/server/runner.rs:56`). Every reply is a 20-byte response header followed by a 4-byte little-endian
status word, and then, for the ops that carry data, a fixed payload. Status `0` means success; a negative
status is one of the errno constants in `src/protocol/errno.rs`.

| Op | Opcode | Request payload | Reply payload after status | Handler |
|---|---|---|---|---|
| `OP_HEALTHCHECK` | `0x0001` | none | none (status only) | `server/handlers/health.rs:19` |
| `OP_CONTROLLER_INFO` | `0x0002` | none | 52-byte register/setup snapshot | `server/handlers/controller_info.rs:25` |
| `OP_IDENTIFY_CONTROLLER` | `0x0003` | none | 88-byte selected identity record | `server/handlers/identify_controller.rs:25` |
| `OP_IDENTIFY_NAMESPACE` | `0x0004` | none | 36-byte selected namespace record | `server/handlers/identify_namespace.rs:25` |
| `OP_SMART_HEALTH` | `0x0005` | none | 177-byte selected health record | `server/handlers/smart_health/handle.rs:29` |
| `OP_CAPACITY` | `0x0006` | none | 8-byte sector count | `server/handlers/capacity.rs:26` |
| `OP_READ_BLOCKS` | `0x0007` | 12-byte `lba(8), sectors(4)` | sector bytes (`sectors * 512`) | `server/handlers/read.rs:27` |
| `OP_WRITE_BLOCKS` | `0x0008` | 12-byte header + sector bytes | none (status only) | `server/handlers/write.rs:22` |
| `OP_FLUSH` | `0x0009` | none | none (status only) | `server/handlers/flush.rs:21` |

The opcodes are defined in `src/protocol/ops.rs:17`. An unrecognised opcode is answered with `E_INVAL`
(`src/server/runner.rs:71`).

Field detail on the data-bearing ops:

- `OP_CONTROLLER_INFO` reads the register block live (`ControllerInfo::read`) and packs `CAP` (u64),
  then `VS`, `CC`, `CSTS`, `AQA`, `INTMS`, `INTMC`, `CMBLOC`, `CMBSZ` (each u32), then the maximum queue
  entries (u16), and finally the timeout units, doorbell stride, min and max page shift, an NVM-supported
  flag, a ready flag, and a fatal flag as bytes (`server/handlers/controller_info.rs:31`). The layout is
  fixed at 52 bytes (`src/protocol/limits.rs:18`).
- `OP_IDENTIFY_CONTROLLER` returns cached fields parsed once at bring-up: vendor and subsystem vendor id,
  the 20-byte serial, the 40-byte model, the 8-byte firmware, version, optional-admin, namespace count,
  MDTS, SQ/CQ entry sizes, optional-NVM, and the volatile-write-cache byte
  (`server/handlers/identify_controller.rs:31`, parsed at `src/admin/identity.rs:35`).
- `OP_IDENTIFY_NAMESPACE` returns the cached NSID-1 record: nsid, size/capacity/used in LBAs, the LBA
  size, metadata size, format index, and formatted-LBA count
  (`server/handlers/identify_namespace.rs:31`, parsed at `src/admin/namespace.rs:43`).
- `OP_SMART_HEALTH` returns the cached SMART snapshot: the critical-warning byte, composite temperature
  in kelvin and in celsius, spare and threshold, percentage-used, endurance-group warning, then ten
  128-bit lifetime counters (data units read/written, host read/write commands, controller busy time,
  power cycles, power-on hours, unsafe shutdowns, media errors, error-log entries), and two 32-bit
  temperature-time counters (`server/handlers/smart_health/handle.rs:35`, parsed at
  `src/admin/health/parse.rs:23`).
- `OP_CAPACITY` returns the namespace sector count, or `E_NODEV` if no IO queue was brought up
  (`server/handlers/capacity.rs:27`).

Errors, all little-endian negative words (`src/protocol/errno.rs`):

```
  E_INVAL    -22   bad op, or a query op carried a payload; also a bad sector count
  E_IO        -5   the NVM read/write/flush command completed with an error
  E_NXIO      -6   the requested LBA range runs past namespace capacity
  E_NODEV    -19   no usable IO queue was created at bring-up
  E_MSGSIZE  -90   the request length does not match the op's fixed layout
```

The read and write handlers validate before touching the device. `rw_parse::parse` rejects a zero
sector count and any count above `MAX_SECTORS` (64) with `E_INVAL`, and rejects `lba + sectors > capacity`
with `E_NXIO`, using a checked add so the bound cannot overflow (`server/handlers/rw_parse.rs:20`). Read
also requires the request payload to be exactly the 12-byte header (`read.rs:33`); write requires the
payload to be the 12-byte header plus exactly `sectors * 512` bytes (`write.rs:33`).

## Architecture and bring-up

The bring-up is a single ordered sequence in `src/setup/sequence.rs:30`. Each step is a broker call or a
controller register step, and a failure returns an `NvmeError` that maps to an exit code, so a partly
built driver never serves IPC.

1. Discover. `find_nvme` calls `mk_device_list` for block-class devices and returns the first PCI
   function whose class/subclass/prog-if is `01/08/02` (NVMe) with an MMIO BAR0 at least `0x4000` bytes
   (`src/discover.rs:32`, `src/constants/pci.rs`). No match is `DeviceNotFound`.
2. Claim. `mk_device_claim` on that device id returns a claim epoch (`src/setup/claim.rs:21`). The epoch
   is the token every later broker call must present, so a stale or revoked claim fails cleanly. See
   [device claim and epochs](../../subsystems/hardware-broker/claim.md).
3. Bus master. `mk_pci_config_write` sets the bus-master bit in the PCI command register
   (`MK_PCI_CFG_COMMAND`, `MK_PCI_CMD_BUS_MASTER`) so the controller can DMA
   (`src/setup/pci.rs:21`). On failure the device is released and the error propagates.
4. Map BAR0. `mk_mmio_map` maps BAR index 0 and returns a user virtual address, length, and grant id
   (`src/setup/mmio.rs:22`). `Regs::new` wraps that address for volatile 32/64-bit register access
   (`src/regs/mmio.rs:25`). See [MMIO grants](../../subsystems/hardware-broker/mmio.md).
5. Bind MSI-X. `mk_irq_bind` with `MK_IRQ_BIND_MSIX` requests one vector (`src/setup/irq.rs:21`). This is
   best-effort: the driver polls every completion and never blocks on the interrupt, so a failed bind
   continues in polling mode with a zero grant (`src/setup/irq.rs:24`). See
   [IRQ grants](../../subsystems/hardware-broker/irq.md).
6. Identify the register block. `ControllerInfo::read` reads `CAP`/`VS`/`CC`/`CSTS` and friends, and
   `is_nvme_register_block` sanity-checks `CAP != 0`, `VS != 0`, and a non-zero max-queue-entries before
   trusting the mapping (`src/setup/sequence.rs:41`, `src/controller/info/is_nvme_register_block.rs:21`).
7. Reset. `reset_to_disabled` writes `CC = 0` and polls `CSTS.RDY` low, bailing on `CSTS.CFS`
   (controller fatal) (`src/admin/controller.rs:24`).
8. Admin queues. `AdminQueue::allocate` maps three DMA regions (submission, completion, and a shared
   4 KiB identify/log scratch buffer) through `mk_dma_map` (`src/admin/queue/allocate.rs:23`).
   `program_registers` writes `AQA` (64 entries each), and the device-side addresses of the admin SQ and
   CQ into `ASQ`/`ACQ` (`src/admin/queue/registers.rs:23`). See
   [DMA grants](../../subsystems/hardware-broker/dma.md).
9. Enable. `enable` requires the controller's minimum page shift to be 12 (4 KiB pages), writes
   `CC.EN | IOSQES=64 | IOCQES=16`, and polls `CSTS.RDY` high (`src/admin/controller.rs:29`).
10. Identify Controller. An Identify (CNS 1) admin command writes 4 KiB into the scratch DMA;
    `ControllerIdentity::parse` extracts the fields (`src/admin/queue/identify_controller.rs:24`,
    `src/admin/command/identify_controller.rs:20`).
11. Identify Namespace. If the controller reports at least one namespace, an Identify (CNS 0) for NSID 1
    is issued and parsed; otherwise the namespace record is `absent()`
    (`src/setup/sequence.rs:53`, `src/admin/namespace.rs:30`).
12. SMART/health. A Get Log Page (LID `0x02`, NSID `0xffffffff`, 512 bytes) fills the scratch DMA and
    `SmartHealth::parse` snapshots it (`src/admin/queue/log.rs:27`, `src/admin/command/get_log_page.rs:20`).
13. IO queue pair. Only if NSID 1 has a 512-byte LBA and a non-zero size does the driver bring up an IO
    queue (`src/setup/sequence.rs:63`). `IoQueue::allocate` maps four DMA regions (SQ, CQ, a 4 KiB PRP
    list, and a 32 KiB data buffer) and computes the SQ/CQ doorbell offsets from `REG_DOORBELL_BASE` and
    the stride (`src/nvm/alloc.rs:24`). `bring_up` then issues Create IO Completion Queue and Create IO
    Submission Queue admin commands for queue id 1 with 8 entries (`src/nvm/setup.rs:23`).

Submission/completion model. A submission entry is a 64-byte `#[repr(C, align(64))]` struct and a
completion entry is 16 bytes; both sizes are asserted at compile time
(`src/admin/command/submission.rs:17`, `src/admin/completion.rs:17`). To submit, the driver writes the
command into the SQ slot at the current tail, advances the tail modulo the queue depth, and rings the SQ
doorbell with the new tail (`src/admin/queue/submit.rs:26`, `src/nvm/submit.rs:25`). To reap, it reads the
CQ slot at the current head and matches the command id and the phase bit; a matched entry advances the
head, flips the phase on wrap, and rings the CQ doorbell (`src/admin/queue/wait.rs:27`,
`src/nvm/wait.rs:26`). Completion success is `(status >> 1) == 0` (`src/admin/completion.rs:33`). The
admin doorbell offsets are fixed at `REG_DOORBELL_BASE` for SQ0 and the next stride step for CQ0
(`src/admin/queue/sq0_tail.rs:19`); the IO doorbell offsets are `REG_DOORBELL_BASE + 2*qid*stride` and
`+1` (`src/nvm/alloc.rs:42`).

DMA and the PRP path. Every buffer the controller touches is a brokered DMA region with a user virtual
address for the driver and a device address for the controller (`src/dma/region.rs:21`). A read or write
copies sector bytes between the IPC buffer and the DMA data region and builds the PRP list: a transfer of
one page uses `prp1` alone, two pages uses `prp1`/`prp2`, and more pages fills the PRP-list DMA with the
remaining page addresses and points `prp2` at it (`src/nvm/prp.rs:20`). The NVM read/write command carries
the LBA, the zero-based block count (`nsectors - 1`), and the PRP pointers
(`src/nvm/transfer.rs:25`, `src/admin/command/nvm_rw.rs:20`). `MAX_SECTORS` is 64, which is 32 KiB and
matches the data DMA region size (`src/nvm/constants.rs:22`).

IRQ handling. The server loop polls the MSI-X grant on every iteration via `mk_irq_poll`, and when the
sequence advances it acknowledges with `mk_irq_ack` (`src/server/runner.rs:75`). The interrupt is only a
wake hint; correctness comes from the completion polling above, which is why a missing MSI-X bind is not
fatal. Both the admin and IO wait loops spin up to a fixed limit and return `ControllerTimeout` if the
controller never completes (`src/admin/queue/constants.rs:22`, `src/nvm/constants.rs:26`).

Teardown. The `Driver` owns a `BrokerHandles` and the DMA regions. On drop, `BrokerHandles` unbinds the
IRQ, unmaps the MMIO grant, and releases the device claim, in that order
(`src/handles/broker_handles_drop.rs:21`), and every `DmaRegion` unmaps itself
(`src/dma/region.rs:46`). Because the server loop never returns, this runs only on an early error exit;
the kernel also revokes every grant tied to the claim when the process dies.

## Protocol and IPC

Two directions cross the capsule boundary. Inbound, clients call the nine ops above. Outbound, the
capsule makes broker syscalls to reach hardware and IPC syscalls to receive and reply.

Client ops (the `NNVM` protocol, `src/protocol/`): `OP_HEALTHCHECK 0x0001`, `OP_CONTROLLER_INFO 0x0002`,
`OP_IDENTIFY_CONTROLLER 0x0003`, `OP_IDENTIFY_NAMESPACE 0x0004`, `OP_SMART_HEALTH 0x0005`,
`OP_CAPACITY 0x0006`, `OP_READ_BLOCKS 0x0007`, `OP_WRITE_BLOCKS 0x0008`, `OP_FLUSH 0x0009`
(`src/protocol/ops.rs:17`). The request is decoded from a 20-byte header and dispatched
(`src/server/runner.rs:54`); replies go to the kernel reply endpoint `0x1_0000_0011` with `mk_ipc_send`
(`src/server/error.rs:26`, `src/protocol/endpoint.rs:18`). The request buffer is sized to the header plus
the maximum read/write payload, so a single receive holds the largest write
(`src/server/runner.rs:30`).

Broker and IPC syscalls the capsule calls (all in `nonos_libc`):

```
  mk_device_list       find NVMe PCI functions           src/discover.rs:34
  mk_device_claim      claim the device, get an epoch     src/setup/claim.rs:22
  mk_pci_config_write  set the bus-master bit             src/setup/pci.rs:22
  mk_mmio_map          map BAR0 registers                 src/setup/mmio.rs:24
  mk_irq_bind          bind one MSI-X vector              src/setup/irq.rs:23
  mk_dma_map           allocate an admin/IO/data DMA      src/dma/region.rs:30
  mk_irq_poll          poll the interrupt sequence        src/server/runner.rs:77
  mk_irq_ack           acknowledge the interrupt          src/server/runner.rs:79
  mk_ipc_recv          receive a client request           src/server/runner.rs:39
  mk_ipc_send          send a reply                       src/server/error.rs:26
  mk_dma_unmap         revoke a DMA grant (Drop)          src/dma/region.rs:48
  mk_irq_unbind        unbind the MSI-X vector (Drop)     src/handles/broker_handles_drop.rs:23
  mk_mmio_unmap        unmap BAR0 (Drop)                  src/handles/broker_handles_drop.rs:24
  mk_device_release    release the claim (Drop)           src/handles/broker_handles_drop.rs:25
```

The broker syscall families and their bounds are documented on the hardware-broker pages: the
[claim](../../subsystems/hardware-broker/claim.md), [MMIO](../../subsystems/hardware-broker/mmio.md),
[DMA](../../subsystems/hardware-broker/dma.md), and [IRQ](../../subsystems/hardware-broker/irq.md)
grants. This capsule is one of the clients those pages describe.

## Security analysis

This driver is different from an application capsule: it holds real hardware authority. Its mask `0xF8019`
grants `Driver`, `DeviceEnum`, `Mmio`, `Irq`, and `Dma` on top of `IPC` and `Memory`
(`Capsule.mk:16`, `src/hardware/nvme_capsule/spawn.rs:51`). Those bits are exactly what the broker checks
before it will claim a device, map a BAR, bind an interrupt, or map DMA (`src/capabilities/types.rs:34`).
So the trust question is not "can it reach hardware" (it can, that is its job) but "how tightly is that
reach bounded".

The broker bounds it in four ways. First, the claim is device-scoped and epoch-gated: the capsule can
only claim one NVMe function, and every MMIO, DMA, and IRQ call must present the claim epoch, so it cannot
act on a device it has not claimed and cannot use a stale claim
(`src/setup/claim.rs:21`, `src/setup/mmio.rs:24`; [claim.md](../../subsystems/hardware-broker/claim.md)).
Second, the MMIO grant is exactly BAR0 of that function at a broker-chosen user address; the driver never
sees a physical address or another device's registers (`src/setup/mmio.rs:22`). Third, each DMA region is
a separate grant with its own device address returned by the broker, and the controller is programmed only
with those broker-issued device addresses (`src/dma/region.rs:30`, `src/admin/queue/registers.rs:23`).
Fourth, every grant is revoked on drop and again by the kernel when the process dies
(`src/handles/broker_handles_drop.rs:21`).

The honest caveat is the absence of an IOMMU on the current target. Bus mastering is enabled
(`src/setup/pci.rs:21`) and the controller is handed device addresses for its queues and data buffer, but
nothing in hardware forces the controller to confine its DMA to those buffers. The trust boundary here is
the device itself: a correct NVMe controller only reads and writes the addresses the driver programmed
into its commands, and the driver only ever programs broker-issued DMA device addresses. A malicious or
buggy controller could DMA outside its buffers, and without an IOMMU the broker cannot prevent that. This
is the same universal DMA caveat that applies to every hardware driver capsule, not something specific to
NVMe.

Within its own boundary the driver is defensive. The request server validates the header magic and
version and rejects anything malformed with `E_INVAL` (`src/protocol/decode.rs:19`,
`src/server/error.rs:29`). The read/write parser bounds the sector count and the LBA range against the
real namespace capacity with a checked add, so a client cannot read or write past the namespace or
overflow the bound (`src/server/handlers/rw_parse.rs:20`). Write requires an exact payload length before
it copies anything into the DMA buffer (`src/server/handlers/write.rs:33`), and both data ops fail with
`E_NODEV` if no IO queue exists (`src/server/handlers/read.rs:29`). There is no panic path: the crate is
`panic = "abort"` and the code returns errno words instead of unwinding (`Cargo.toml:26`).

Isolation from other drivers and from the rest of the system is the kernel's, not the driver's. This is a
CPL 3 process that only speaks its `NNVM` protocol over IPC and reaches its one claimed device through the
broker. Another storage driver (for example the AHCI capsule spawned alongside it in the same storage
plan, `src/userspace/init/spawn_plan/drivers_storage.rs:17`) claims a different device with a different
epoch and cannot touch this one. A client that wants block I/O must hold the capability to reach
`driver.nvme0` and speak the protocol; it never gets a handle to the controller.

## How to contribute

The source lives at `userland/capsule_driver_nvme/`. The layout is one concern per subtree: `src/setup/`
is the bring-up sequence and the broker calls, `src/admin/` is the admin queue and its commands,
`src/nvm/` is the IO queue and the read/write/flush path, `src/controller/` decodes the register block,
`src/dma/` and `src/handles/` wrap the broker grants, `src/protocol/` is the wire format, and
`src/server/` is the request loop and the per-op handlers.

To add a client op:

1. Add the opcode constant to `src/protocol/ops.rs:17` and, if it carries a fixed payload, its length to
   `src/protocol/limits.rs:17`.
2. Write the handler as one file under `src/server/handlers/`, exposing
   `pub fn handle(driver: ..., req: &Request, ..., tx: &mut [u8])` that encodes the response header,
   writes the status word, and sends with `mk_ipc_send`, following `capacity.rs` or `read.rs`. Re-export
   it from `src/server/handlers/mod.rs:17`.
3. Wire it into the dispatch match in `src/server/runner.rs:54`, adding the op to the zero-payload guard
   at `runner.rs:56` if it takes no payload.

To add an NVMe command, add a `Submission` constructor under `src/admin/command/` (one file per command,
like `nvm_flush.rs`), then an `AdminQueue` or `IoQueue` method under `src/admin/queue/` or `src/nvm/`
that submits it and waits on the completion.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `Capsule.mk:18`):

```
  make nonos-mk-driver-nvme            build the capsule ELF
  make nonos-mk-driver-nvme-sign       produce the id cert, manifest, and attestation trailer
  make nonos-mk-driver-nvme-verify     verify the signed artifacts against the trust anchor
  make nonos-mk-check-driver-nvme-keys check the per-capsule signing keys exist
```

For a kernel image that embeds and spawns the driver, `make nonos-mk-driver-nvme-prod` builds the
`microkernel-driver-nvme` profile with the signed NVMe artifacts baked in (`Makefile:1015`). The README
also records the direct build and gate commands (`make -B nonos-mk-driver-nvme`,
`bash nonos-ci/run-static-checks.sh`) (`userland/capsule_driver_nvme/README.md:165`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every path returns an `NvmeError` or an errno word, and the release profile is
`panic = "abort"`, `Cargo.toml:26`); modular files, one unit per file, with `mod.rs` used only for
re-exports; and the AGPL header at the top of every source file, matching the header on every existing
module. Every setup phase must have reverse-order rollback, which is what the `Drop` impls provide
(`userland/capsule_driver_nvme/README.md:130`).

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[DRIVER-NVME] capsule spawned` from the storage spawn plan (tag `DRIVER-NVME`, message
`capsule spawned`) (`src/userspace/init/spawn_plan/drivers_storage.rs:50`,
`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line means the
capsule never started, usually a signature, manifest, or capability failure; the error path prints an
`[ERROR]` line instead (`src/userspace/init/capsule_boot/run.rs:32`).

Bring-up failure modes map to distinct process exit codes (`src/error/types.rs:30`), which is the fastest
way to tell how far setup got:

- `DeviceNotFound` (exit 30). No PCI function matched the NVMe class/subclass/prog-if `01/08/02` with a
  large-enough MMIO BAR0 (`src/discover.rs:49`). On QEMU this means no `-device nvme`; on real hardware it
  means the controller was not enumerated as expected.
- `ClaimFailed` (exit 31). `mk_device_claim` was refused, usually a missing `Driver` capability or a
  device already claimed by another capsule (`src/setup/claim.rs:23`).
- `BrokerCallFailed` (exit 32). A bus-master write, MMIO map, or DMA map was refused by the broker
  (`src/setup/pci.rs:23`, `src/setup/mmio.rs:25`, `src/dma/region.rs:31`).
- `UnsupportedController` (exit 33). The mapped register block did not look like NVMe, or the controller
  raised `CSTS.CFS` (controller fatal) during reset or enable
  (`src/controller/info/is_nvme_register_block.rs:21`, `src/admin/controller.rs:41`). This is the "controller
  not ready" case.
- `UnsupportedPageSize` (exit 34). The controller's minimum page shift is not 12; the driver only supports
  4 KiB pages (`src/admin/controller.rs:30`).
- `ControllerTimeout` (exit 35). A ready poll or a queue completion never landed within the fixed spin
  limit (`src/admin/controller.rs:49`, `src/admin/queue/wait.rs:36`, `src/nvm/wait.rs:35`). This is the
  DMA/completion timeout case, and on real hardware it usually points at bus mastering, an MSI-X or
  addressing issue, or a controller that hung.
- `AdminCommandFailed` (exit 36). An admin or IO command completed with a non-zero status field
  (`src/admin/queue/wait.rs:32`, `src/nvm/wait.rs:31`).

Runtime symptoms after a successful boot:

- Every read, write, flush, or capacity call returns `E_NODEV`. No IO queue was brought up, because NSID 1
  did not report a 512-byte LBA with a non-zero size at setup (`src/setup/sequence.rs:63`). Identify and
  SMART still answer, so `OP_IDENTIFY_NAMESPACE` is the probe: an absent namespace shows nsid 0. This is
  the "no namespaces" case.
- A read/write returns `E_NXIO` or `E_INVAL`. The requested range runs past capacity (`E_NXIO`), or the
  sector count is zero or above 64 (`E_INVAL`) (`src/server/handlers/rw_parse.rs:26`). Check the reported
  capacity with `OP_CAPACITY` first.
- A read/write returns `E_MSGSIZE`. The payload length did not match the fixed layout: 12 bytes for a
  read request, 12 bytes plus `sectors * 512` for a write (`src/server/handlers/read.rs:33`,
  `write.rs:33`).
- A read/write/flush returns `E_IO`. The command reached the controller but completed with an error
  status (`src/server/handlers/read.rs:40`, `flush.rs:27`).

## Source map

```
  src/main.rs                          _start -> setup::run -> server::run
  src/setup/sequence.rs                the whole bring-up: claim, bus master, map, reset, enable, identify, IO queue
  src/setup/{claim,pci,mmio,irq}.rs    the individual broker calls for claim / bus master / BAR0 / MSI-X
  src/discover.rs                      mk_device_list scan for the NVMe PCI function
  src/handles/                         BrokerHandles: device id, MMIO grant, IRQ grant, and their Drop teardown
  src/dma/region.rs                    DmaRegion: mk_dma_map wrapper with user_va, device_addr, and Drop unmap
  src/regs/mmio.rs                     Regs: volatile 32/64-bit access over the BAR0 mapping
  src/constants/                       register offsets, CC/CSTS bits, CAP field decoders, PCI class
  src/controller/info/                 ControllerInfo: read the register block and decode CAP fields
  src/admin/queue/                     AdminQueue: allocate, program AQA/ASQ/ACQ, submit, wait, identify, log
  src/admin/command/                   Submission builders (identify, create IO cq/sq, get log page, nvm rw, flush)
  src/admin/{identity,namespace,health}/  parsers for Identify Controller, Identify Namespace, SMART
  src/nvm/                             IoQueue: allocate, create queues, PRP build, transfer, flush, submit, wait
  src/protocol/                        the NNVM wire format: header, ops, errno, limits, decode/encode
  src/server/runner.rs                 the request loop: poll IRQ, receive, decode, dispatch
  src/server/handlers/                 one file per op (health, controller_info, identify_*, smart_health, capacity, read, write, flush)
  src/error/types.rs                   NvmeError and the exit-code mapping
  Capsule.mk                           slug, handle, ports, capability mask, kernel mirror
  src/hardware/nvme_capsule/           the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/drivers_storage.rs  the storage-driver spawn entry
  nonos-mk/capsule.mk                  the generated nonos-mk-driver-nvme[-sign|-verify] targets
```

Every reference above is verified against those trees.

# capsule_driver_virtio_blk (full reference)

`capsule_driver_virtio_blk` is the virtio block-device backend: the userspace driver that owns a
virtio-blk PCI device, drives its virtqueue over brokered DMA, and serves sector-oriented read, write,
and flush requests over IPC. It is the default disk under QEMU, where `-device virtio-blk-pci` is the
backing drive the image is built against (`Makefile:272`). It owns no filesystem, no partition table, and
no cache; every byte of interpretation above a raw sector lives in a capsule layered on top of it.

The kernel spawns it under service handle `driver.virtio_blk0` on service port 4202 with a reply port on
4203, and its capability mask is `0x1F8019` (`userland/capsule_driver_virtio_blk/Capsule.mk:16`). The
source is `userland/capsule_driver_virtio_blk/`.

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

The capsule is a `no_std`/`no_main` binary. `_start` initialises the heap, then loops calling `setup::run`
until it returns a live `Driver`, yielding 64 times between attempts so a device that is not yet claimable
does not spin the CPU; once bring-up succeeds it enters `server::run`, which never returns
(`userland/capsule_driver_virtio_blk/src/main.rs:30`). There is no policy in the loop: the driver
discovers exactly one virtio-blk device, negotiates it, allocates its DMA regions, probes capacity, and
then answers IPC.

Everything above a sector is out of scope by design. The driver exposes read, write, and flush against
LBA ranges and reports capacity; it does not parse a partition table, mount a filesystem, keep a
writeback cache, or hold block data past the reply that carried it (`README.md:44`, `README.md:123`).
Storage and filesystem capsules sit above it and hold the interpretation. This split is the reason the
driver's authority is hardware-facing and narrow: it needs to touch one device and move bytes, nothing
more.

## Identity

Everything the kernel and the broker need to name and reach the driver comes from its `Capsule.mk` and its
kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-virtio-blk` | `Capsule.mk:6` |
| Service handle | `driver.virtio_blk0` | `Capsule.mk:7`, `src/hardware/virtio_blk_capsule/spawn.rs:32` |
| Namespace | `systems.nonos.driver.virtio_blk0` | `Capsule.mk:12` |
| Service endpoint | `service:4202:driver.virtio_blk0` | `Capsule.mk:13`, `spawn.rs:33` |
| Reply endpoint | `reply:4203:endpoint.4294967304` | `Capsule.mk:14`, `spawn.rs:34` |
| Binary name | `driver_virtio_blk` | `Capsule.mk:10` |
| Capability mask | `0x1F8019` | `Capsule.mk:16` |
| Kernel mirror | `src/hardware/virtio_blk_capsule` | `Capsule.mk:17` |

The reply endpoint id `4294967304` is `0x1_0000_0008`, the same constant the driver hard-codes as its
outbound `KERNEL_REPLY_ENDPOINT` for every reply it sends
(`src/protocol/endpoint.rs:16`).

The mask `0x1F8019` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x000008  IPC          bit()       8   types.rs:59
  0x000010  Memory       bit()      16   types.rs:60
  0x008000  DeviceEnum   bit()   32768   types.rs:71
  0x010000  Driver       bit()   65536   types.rs:72
  0x020000  Mmio         bit()  131072   types.rs:73
  0x040000  Irq          bit()  262144   types.rs:74
  0x080000  Dma          bit()  524288   types.rs:75
  0x100000  Pio          bit() 1048576   types.rs:76
  ------
  0x1F8019  = 8 + 16 + 32768 + 65536 + 131072 + 262144 + 524288 + 1048576
```

The kernel spawn path requests exactly those eight capabilities and no others, assembled by OR-ing the
same bits (`src/hardware/virtio_blk_capsule/spawn.rs:51`). There is no `CoreExec` bit (1), no `Network`
bit (4), and no `FileSystem` bit (64). The mask is the hardware-driver envelope: it can enumerate
devices, claim one, map its registers by MMIO or PIO, bind its interrupt, allocate DMA buffers, and speak
IPC. That is precisely the set the broker paths below check for, and nothing in the mask lets the driver
read another device, open a socket, or reach a filesystem.

## Operation reference

The server receives a message, decodes the 20-byte header, and dispatches on the opcode
(`src/server/runner.rs:45`). Every request begins with the `NBLK` header: magic `0x4E42_4C4B`, version
`1`, then op, flags, a reserved `u16`, a `request_id`, and a `payload_len`, laid out across 20 bytes
(`src/protocol/header.rs:16`, `src/protocol/decode.rs:17`). A header that is short, has the wrong magic,
or the wrong version is rejected and answered with `E_INVAL` (-22) and a zeroed request stub
(`src/server/runner.rs:39`, `src/server/error.rs:25`). Every reply begins with the same 20-byte response
header followed by a 4-byte little-endian status word; ok is `0`, and the error codes are `E_INVAL` (-22),
`E_IO` (-5), `E_MSGSIZE` (-90), and `E_NXIO` (-6) (`src/protocol/errno.rs:16`,
`src/protocol/encode.rs:17`).

The five opcodes are defined in one file (`src/protocol/ops.rs:16`):

| Op | Opcode | Request body | Reply payload | Handler |
|---|---|---|---|---|
| `OP_HEALTHCHECK` | 1 | none | status word only | `handlers/health.rs:18` |
| `OP_CAPACITY` | 2 | none | status word + 8-byte sector count | `handlers/capacity.rs:22` |
| `OP_READ_BLOCKS` | 3 | 12-byte block header | status word + read bytes | `handlers/read/handle.rs:24` |
| `OP_WRITE_BLOCKS` | 4 | 12-byte block header + data | status word only | `handlers/write/handle.rs:23` |
| `OP_FLUSH` | 5 | none | status word only | `handlers/flush.rs:21` |

An unknown opcode is answered with `E_INVAL` (`src/server/runner.rs:51`).

### OP_HEALTHCHECK (1)

Liveness only. The handler writes status `0` and replies; it touches neither the device nor the queue, so
a reply proves the server loop is running (`src/server/handlers/health.rs:18`).

### OP_CAPACITY (2)

Reports the device's sector count. The handler writes status `0` then appends the driver's cached
`capacity_sectors` as 8 little-endian bytes, for a payload of `STATUS_LEN + CAPACITY_PAYLOAD_LEN` = 12
bytes after the header (`src/server/handlers/capacity.rs:22`, `src/protocol/limits.rs:21`). The capacity
is read once at bring-up from the legacy config space and cached; it is not re-read per request
(`src/setup/sequence.rs:50`).

### OP_READ_BLOCKS (3)

The 12-byte block header is `lba` (u64 at offset 0) then `nsectors` (u32 at offset 8)
(`src/server/handlers/read/request.rs:36`). The parser enforces the bounds before any DMA: the body must
be at least `READ_REQ_LEN` (12) and `payload_len` must equal 12 (`E_MSGSIZE`); `nsectors` must be non-zero
and at most `MAX_SECTORS_PER_REQUEST` = 64 (`E_INVAL`); and `lba + nsectors` must not exceed
`capacity_sectors` (`E_NXIO`), with the addition itself `checked_add` so an overflow is `E_INVAL`
(`src/server/handlers/read/request.rs:33`, `src/constants/queue.rs:22`). On a valid request the driver
submits a read to the device, then copies `nsectors * 512` bytes out of the DMA data buffer into the reply
after the status word (`src/server/handlers/read/handle.rs:32`, `src/server/handlers/read/reply.rs:22`).
Device `IOERR` maps to `E_IO`, an unsupported request to `E_INVAL` (`handlers/read/handle.rs:42`).

### OP_WRITE_BLOCKS (4)

Same 12-byte header, followed by the data to write. The parser requires the body to be exactly
`RW_HEADER_LEN + nsectors * 512` bytes and `payload_len` to match, else `E_MSGSIZE`; the `nsectors` and
capacity bounds are identical to read (`src/server/handlers/write/request.rs:42`). The handler copies the
body's data region into the DMA buffer, submits a write, and replies with the status word alone
(`src/server/handlers/write/handle.rs:31`). Ok is `0`, `IOERR` is `E_IO`, unsupported is `E_INVAL`.

### OP_FLUSH (5)

Forces a device flush. The handler submits a flush with `lba` and `nsectors` both zero and replies with
the status word (`src/server/handlers/flush.rs:22`). Flush is only meaningful if the device advertised the
flush feature at negotiation; a device that reports the request unsupported yields `E_INVAL`.

## Architecture and bring-up

Bring-up is an ordered transaction in `setup::run` (`src/setup/sequence.rs:23`). Each step depends on the
grant the previous one produced, and each failure rolls back the grants already taken, so a failed bring-up
leaves the device unclaimed and no grants leaked.

1. **Discover.** `find_virtio_blk` lists broker devices and matches on virtio vendor `0x1AF4` with device
   id `0x1001` (transitional) or `0x1042` (modern) on a PCI bus, skipping any with no interrupt pin or an
   unrouted line, and picks the first PIO or MMIO register BAR (`src/discover.rs:27`, `src/constants/pci.rs:16`).
2. **Claim.** `mk_device_claim` takes exclusive ownership and returns the claim epoch that every later
   grant must quote (`src/setup/claim.rs:17`). The epoch is the broker's anti-stale linchpin: a grant
   quoting an old epoch after a release is rejected with `StaleEpoch` ([claim](../../subsystems/hardware-broker/claim.md)).
3. **Registers.** `registers::grant` maps the register BAR: an MMIO BAR through `mk_mmio_map` (page-rounded
   length), or a port BAR through `mk_pio_grant`, wrapped in a `RegisterGrant` enum so the rest of the
   driver reads registers the same way regardless of BAR kind (`src/setup/registers.rs:42`,
   `src/regs/state/r32.rs:23`). Register access is uncached device memory, never RAM
   ([mmio](../../subsystems/hardware-broker/mmio.md)).
4. **IRQ.** `irq::bind` binds the device interrupt, trying legacy INTx on the discovered line first and
   falling back to MSI-X; a total failure releases the register and device grants before returning
   (`src/setup/irq.rs:19`) ([irq](../../subsystems/hardware-broker/irq.md)).
5. **DMA.** Three `mk_dma_map` calls allocate the three device-visible regions in order, queue then header
   then data, each rolling back all prior grants on failure: the virtqueue ring (`VQ_REGION_SIZE` = 16384
   bytes), the request header buffer (`HEADER_BUF_LEN` = 4096 bytes), and the data buffer (`DATA_BUF_LEN` =
   64 * 512 = 32768 bytes) (`src/setup/dma/map_queue.rs:20`, `src/setup/dma/map_header.rs:20`,
   `src/setup/dma/map_data.rs:20`, `src/constants/queue.rs:26`). Each region carries a device physical
   address the device programs into its descriptors, allocated and zeroed by the broker before the capsule
   sees it ([dma](../../subsystems/hardware-broker/dma.md)).
6. **Negotiate.** `bring_up` walks the legacy virtio status handshake: reset, `ACKNOWLEDGE`, `DRIVER`,
   read host features and offer back only `VIRTIO_BLK_F_FLUSH` (bit 9) if the device advertised it,
   `FEATURES_OK` (re-checked, or `FAILED` and abort), select queue 0, read its max size (rejecting a queue
   below 3 or above the 256 supported), write the queue PFN, then `DRIVER_OK`
   (`src/init.rs:25`, `src/constants/status.rs:16`, `src/constants/regs.rs:16`, `src/queue/layout.rs:59`).
7. **Probe and arm.** Read `capacity_sectors` from the legacy config capacity register at offset `0x14`
   (rejecting zero capacity) and acknowledge the initial IRQ, then hand back a `Driver` holding the IRQ
   grant id, the queue, the register handle, and the capacity (`src/setup/sequence.rs:50`,
   `src/setup/driver.rs:18`).

### The virtqueue and descriptor chain

The queue region is zeroed and laid out as a legacy split virtqueue: the descriptor table at offset 0, the
available ring after it, and the used ring page-aligned after that (`src/queue/layout.rs:31`). The driver
uses a single request in flight, so it always builds the chain from descriptor 0.

`post_request` writes the request header, builds the descriptor chain, and publishes it
(`src/queue/post.rs:23`). The header is a 16-byte virtio-blk request header at header-buffer offset 0
(type, a reserved u32, the LBA) with the status byte at offset 64 pre-set to `0xFF`
(`src/queue/post/header.rs:21`, `src/constants/queue.rs:23`). The chain is header, optional data, status
(`src/queue/post/descriptors.rs:27`):

- Descriptor 0 points at the 16-byte header with `NEXT`, chaining to descriptor 1.
- For read and write, descriptor 1 points at the data buffer, sized `nsectors * 512`, with `NEXT`; read
  additionally sets `WRITE` so the device fills it, write leaves it device-readable
  (`src/queue/post/descriptors.rs:45`). Flush omits the data descriptor entirely and chains the header
  straight to status (`descriptors.rs:32`).
- The last descriptor points at the 1-byte status, `WRITE` so the device writes it.

`publish_avail` bumps the available ring index by one to hand the chain to the device
(`src/queue/post/publish.rs:19`).

### The DMA and completion path

`submit` posts the chain, writes the queue-notify register to kick the device, then waits for completion
(`src/io/submit.rs:27`). The wait is IRQ-driven with a used-ring fallback: it snapshots the interrupt
sequence via `mk_irq_poll`, then loops until either the used ring index advances to the expected value or
the interrupt sequence changes, yielding between polls and giving up with `Timeout` after `MAX_YIELDS` =
200000 spins (`src/io/submit.rs:40`, `src/io/read_seq.rs:19`, `src/queue/used.rs:21`). On completion it
reads the device's status byte, acknowledges the IRQ with `mk_irq_ack`, and maps `VIRTIO_BLK_S_OK` to
success, `S_IOERR` to `Io`, and `S_UNSUPP` to `Unsupported` (`src/io/submit.rs:53`,
`src/constants/request.rs:19`). The data buffer read back by a read is bounded to the buffer length before
it is copied out, so an over-large length cannot walk past the mapping (`src/queue/used.rs:27`).

Teardown mirrors bring-up in reverse. The rollback helpers unmap DMA, unbind the IRQ, release the register
grant, and release the device claim, in that order, at every failed bring-up step
(`src/setup/dma/rollback.rs:18`). The broker also revokes every grant automatically on process exit
through `release_all_for_pid`, so a crash leaks nothing ([claim](../../subsystems/hardware-broker/claim.md)).

## Protocol and IPC

The capsule serves one endpoint and speaks two IPC syscalls. It receives requests with `mk_ipc_recv` into
a buffer sized `HDR_LEN + MAX_RW_PAYLOAD_BYTES` (20 + 32768), and sends every reply with `mk_ipc_send` to
the fixed `KERNEL_REPLY_ENDPOINT` `0x1_0000_0008` (`src/server/runner.rs:31`, `src/server/error.rs:23`).
The client-facing opcodes and their bounds are the [operation reference](#operation-reference) above.

The broker syscalls the driver calls, all through `nonos_libc`, and the grant class each belongs to:

```
  discovery   mk_device_list      enumerate broker devices        src/discover.rs:29
  claim       mk_device_claim     take the device, get the epoch   src/setup/claim.rs:18
              mk_device_release   drop the claim (rollback)        src/setup/registers.rs:54
  mmio/pio    mk_mmio_map         map an MMIO register BAR         src/setup/registers.rs:52
              mk_mmio_unmap       unmap it (rollback)              src/setup/registers.rs:37
              mk_pio_grant        grant a port BAR                 src/setup/registers.rs:63
              mk_pio_release      release it (rollback)            src/setup/registers.rs:38
              mk_pio_read/write   port in/out through the grant    src/regs/pio.rs:17
  irq         mk_irq_bind         bind INTx or MSI-X               src/setup/irq.rs:21
              mk_irq_poll         read the interrupt sequence      src/io/read_seq.rs:21
              mk_irq_ack          acknowledge an interrupt         src/io/submit.rs:54
              mk_irq_unbind       unbind it (rollback)             src/setup/dma/rollback.rs:19
  dma         mk_dma_map          allocate a device-visible buffer src/setup/dma/map_queue.rs:27
              mk_dma_unmap        free it (rollback)               src/setup/dma/rollback.rs:36
  scheduling  mk_yield            yield while waiting              src/io/submit.rs:47
```

The register handshake itself is not IPC: once the register BAR is mapped, `bring_up` and `submit` touch
the virtio registers by volatile MMIO reads and writes, or by `mk_pio_read`/`mk_pio_write` if the BAR is a
port BAR, through the `Regs` abstraction (`src/init.rs:26`, `src/regs/state/r32.rs:23`). The virtqueue
mechanics, the descriptor chain and available-ring publish, are direct volatile writes into the
broker-allocated DMA regions, and the device is kicked by a single write to the queue-notify register
(`src/io/submit.rs:36`).

## Security analysis

The virtio-blk driver is a hardware-facing capsule, so its authority is real device authority, but it is
bounded on every axis the broker controls. Its mask `0x1F8019` grants IPC, Memory, DeviceEnum, Driver,
Mmio, Irq, Dma, and Pio, and nothing else (`Capsule.mk:16`,
`src/hardware/virtio_blk_capsule/spawn.rs:51`). It has no CoreExec, no Network, and no FileSystem
capability. It cannot spawn a process, open a socket, or reach a filesystem; it can touch one claimed
device and move bytes over IPC.

Every hardware action goes through the broker, which validates it against the claim and the epoch. The
claim is exclusive: only one capsule can hold a virtio-blk device at a time, so nothing else can be mapping
its BARs or taking its interrupts underneath it ([claim](../../subsystems/hardware-broker/claim.md)). The
MMIO grant can only map memory inside the device's own BAR, never an arbitrary physical address, because
the physical base comes from the kernel's device table and not from the request, and the grant excludes
the device's MSI-X table ([mmio](../../subsystems/hardware-broker/mmio.md)). The IRQ grant delivers the
device interrupt on a kernel-owned vector; the capsule waits and acknowledges through syscalls and never
touches the interrupt controller ([irq](../../subsystems/hardware-broker/irq.md)). The DMA regions are
allocated and zero-scrubbed by the broker before the capsule sees them, so a buffer never hands the device
a previous tenant's bytes, and the request size is capped at the BLOCK class ceiling of 1024 pages, so a
misbehaving driver cannot exhaust physical memory through the DMA path
([dma](../../subsystems/hardware-broker/dma.md)).

The request bounds are the driver's own line of defence against a malicious client. Read and write both
require `nsectors` in `1..=64`, a `payload_len` that exactly matches the header-plus-data size, and an
`lba + nsectors` that stays within the probed capacity, with the addition `checked_add` so a wrapped LBA
is rejected rather than truncated (`src/server/handlers/read/request.rs:33`,
`src/server/handlers/write/request.rs:42`). A read never copies more than the DMA buffer length back into
the reply (`src/queue/used.rs:27`). So a client cannot make the driver DMA outside its data buffer, read a
sector past the device, or overrun the reply.

The honest caveat is the IOMMU. NONOS carries an `IommuDomain` abstraction but its hardware backend is
behind the `nonos-arch-iommu` feature and is not engaged in the shipping builds, so the device physical
address the broker hands back is a raw physical address and a *malicious or buggy device* could in
principle DMA to any physical address regardless of the grant
([dma](../../subsystems/hardware-broker/dma.md)). The broker bounds what the *capsule* may allocate and
program; it does not yet bound what the device does once running. Block-device DMA safety therefore rests
on the software bounds above plus the assumption of non-malicious device hardware, and enabling the IOMMU
backend is the path to closing that gap.

Isolation from other capsules is the kernel's, not the driver's: it is a CPL 3 user binary that speaks
only its endpoint and its brokered device, verified and enrolled at spawn like every other capsule. On the
kernel side, the in-kernel client that talks to this endpoint is itself gated: only a caller holding
`CAP_DRIVER` may reach the block surface (`src/hardware/virtio_blk_capsule/capability.rs:30`).

## How to contribute

The source lives at `userland/capsule_driver_virtio_blk/`. The layout is one concern per subtree:
`src/discover.rs` finds the device, `src/setup/` is the bring-up transaction and its rollback,
`src/init.rs` is the virtio handshake, `src/queue/` builds and reads the virtqueue, `src/io/` submits and
waits, `src/regs/` abstracts MMIO versus PIO register access, `src/protocol/` is the wire format, and
`src/server/` is the request loop and the per-op handlers. Constants are grouped under `src/constants/`.

To change or add an operation:

1. Add the opcode in `src/protocol/ops.rs:16` and re-export it from `src/protocol/mod.rs:31`.
2. Write the handler as its own module under `src/server/handlers/`, one file per op (or a subdirectory
   with `handle.rs`, `request.rs`, and `reply.rs` split out, as `read/` and `write/` are). Parse and
   bounds-check the body before any DMA, returning a negative errno on rejection the way
   `src/server/handlers/read/request.rs` does, and reply through `reply_with_status` or a payload-carrying
   reply.
3. Wire the opcode into the dispatch match in `src/server/runner.rs:45`, and re-export the handler from
   `src/server/handlers/mod.rs`.
4. If it needs a new virtqueue direction or descriptor shape, extend `Direction`
   (`src/queue/post/direction.rs:18`) and the chain builder (`src/queue/post/descriptors.rs:27`); keep the
   header and status descriptor layout intact.

The kernel-side mirror at `src/hardware/virtio_blk_capsule/` carries a matching protocol definition and a
client; the header, ops, and errno modules there are documented as mirrors of the userland source and must
be kept in sync (`src/hardware/virtio_blk_capsule/protocol/header.rs:20`).

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_driver_virtio_blk/Capsule.mk:19`):

```
  make nonos-mk-driver-virtio-blk              build the capsule ELF
  make nonos-mk-driver-virtio-blk-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-driver-virtio-blk-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-driver-virtio-blk-keys   check the per-capsule signing keys exist
```

The README documents `make -B nonos-mk-driver-virtio-blk` as the build invocation and
`bash nonos-ci/run-static-checks.sh` as the static gate (`README.md:130`). For a running kernel that
includes the driver, `make nonos-mk-driver-virtio-blk-prod` builds the kernel with the
`microkernel-driver-virtio-blk` feature and the driver's signed artifacts (`Makefile:940`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns errors as a negative status word, never a panic; the
release profile is `panic = "abort"`, `Cargo.toml:26`); modular files, one unit per file, with `mod.rs`
used only for re-exports (as `src/protocol/mod.rs` and `src/server/handlers/mod.rs` are); and the AGPL
header at the top of every source file, matching the header on every existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[DRIVER-VIRTIO-BLK] capsule spawned`, emitted by the boot path once `spawn_driver_virtio_blk_capsule`
returns ok (`src/userspace/init/spawn_plan/drivers_virtio_io.rs:39`,
`src/userspace/init/capsule_boot/run.rs:29`). An absent line means the capsule never started, usually a
signature, manifest, or capability failure; the error path prints the failure instead
(`src/userspace/init/capsule_boot/run.rs:32`).

Failure modes and where to look:

- Bring-up never completes. `_start` retries `setup::run` forever, so a driver that spawned but serves
  nothing is stuck in bring-up (`src/main.rs:34`). The usual causes are broker refusals: the claim is
  taken by another instance, or the device is not in the broker table at all. The broker narrates each on
  the console, and a `NONOS_DEVICE_CENSUS=1` build renders the device table so you can confirm the device
  is present before any driver runs ([claim](../../subsystems/hardware-broker/claim.md)). A DMA refusal
  prints a `[DMA]` line naming the failed check ([dma](../../subsystems/hardware-broker/dma.md)).
- Bring-up aborts on negotiation. `bring_up` returns a specific string for each virtio failure:
  `features-ok rejected`, `requestq missing`, or `unsupported requestq size` (`src/init.rs:39`), and
  `setup::run` returns `no virtio-blk device` or `virtio-blk: zero capacity` (`src/setup/sequence.rs:24`).
  Zero capacity means the config register read back zero, which points at the device model, not the driver.
- A request returns an error status. The status word in the reply is the diagnosis. `E_MSGSIZE` (-90)
  means the body length or `payload_len` did not match the declared sector count; `E_INVAL` (-22) means a
  bad opcode, a zero or over-64 sector count, an unsupported device request, or a header that failed to
  decode; `E_NXIO` (-6) means the LBA range ran past the device capacity; `E_IO` (-5) means the device
  reported an I/O error or the completion wait failed (`src/protocol/errno.rs:16`,
  `src/server/handlers/read/request.rs:33`).
- A request hangs. `submit` gives up with `Timeout` after 200000 yields waiting for the used ring or a new
  interrupt sequence (`src/io/submit.rs:44`); a persistent timeout points at the IRQ binding or the device
  not completing, not at the request parsing.

## Source map

```
  src/main.rs                         _start: heap init, retry setup::run, then server::run
  src/discover.rs                     find_virtio_blk: match vendor/device, pick register BAR
  src/setup/sequence.rs               the bring-up transaction (claim, regs, irq, dma, negotiate, probe)
  src/setup/{claim,registers,irq}.rs  the claim, register (MMIO/PIO), and IRQ grants
  src/setup/dma/                      the three DMA regions and the ordered rollback chain
  src/setup/driver.rs                 Driver: irq grant, queue, regs, capacity
  src/init.rs                         bring_up: the legacy virtio status handshake and feature negotiation
  src/queue/layout.rs                 the split-virtqueue layout and Queue state
  src/queue/post/                     header, descriptor chain, and available-ring publish
  src/queue/used.rs                   used-ring index, status byte, and bounded data access
  src/io/submit.rs                    submit: notify, IRQ-poll wait with used-ring fallback, ack
  src/io/read_seq.rs                  mk_irq_poll wrapper for the interrupt sequence
  src/regs/                           MMIO vs PIO register access behind the Regs abstraction
  src/protocol/                       the NBLK header, ops, errno, limits, and reply endpoint
  src/server/runner.rs                the receive loop and opcode dispatch
  src/server/handlers/                health, capacity, read, write, flush handlers
  src/constants/                      pci, queue, regs, request, and status constants
  Capsule.mk                          slug, handle, ports, capability mask, kernel mirror
  src/hardware/virtio_blk_capsule/    the kernel-side embed, verified spawn, and mirrored protocol
  src/userspace/init/spawn_plan/drivers_virtio_io.rs   the driver spawn entry and boot marker
  src/capabilities/types.rs           the capability bit definitions the mask decomposes against
  nonos-mk/capsule.mk                 the generated nonos-mk-driver-virtio-blk[-sign|-verify] targets
```

Every reference above is verified against those trees.

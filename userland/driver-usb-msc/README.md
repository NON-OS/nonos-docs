# capsule_driver_usb_msc (full reference)

`capsule_driver_usb_msc` is the USB Mass Storage class driver in the NONOS tree. It is a class capsule
that sits above the xHCI host-controller driver: it classifies USB configuration descriptors, extracts
the bulk-in and bulk-out endpoints of a SCSI-transparent Bulk-Only Transport interface, and builds the
BOT command block wrappers and SCSI command blocks that the storage path needs. It owns class framing
only. It does not touch a controller register, an interrupt, a DMA buffer, or an I/O port, and it does
not move a single byte of block data itself. This is the exhaustive reference; the short version lives in
[storage-capsules.md](../storage-capsules.md) section 5 and the driver list in
[drivers.md](../drivers.md).

Be honest about scope up front: this is a real, compiling, signed, kernel-spawnable capsule, but it is an
early slice. It builds and validates the BOT/SCSI wire; it does not schedule any USB transfer, and it
does not yet publish a block device. The [current implemented surface](#implemented-versus-stub) section
below draws that line exactly.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [Architecture and bring-up](#architecture-and-bring-up)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [Implemented versus stub](#implemented-versus-stub)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The capsule is a `no_std`/`no_main` userland binary. `_start` initializes the heap and calls
`server::run`, which never returns (`userland/capsule_driver_usb_msc/src/main.rs:32`,
`src/main.rs:36`). The server is a single-threaded request loop: it receives one IPC message on the
service inbox, parses the shared `NUMS` header, dispatches to a handler by opcode, and replies to the
sender's pid (`src/server/runner.rs:34`, `src/server/dispatch.rs:21`, `src/server/respond.rs:21`).

Its job is class framing. A block-storage caller first hands it a raw USB configuration descriptor; the
capsule walks the descriptor records, finds an interface with class `0x08`, subclass `0x06`, protocol
`0x50` (mass storage, SCSI-transparent, Bulk-Only Transport), and records that interface's bulk-in and
bulk-out endpoint addresses (`src/descriptors/wire.rs:20`, `src/descriptors/visitor.rs:34`). After that,
the caller asks for BOT command wrappers for INQUIRY, READ CAPACITY(10), READ(10), and WRITE(10); the
capsule builds the 31-byte command block wrapper with a fresh monotonic tag and returns it
(`src/server/handlers/build_inquiry.rs:24`, `src/bot/cbw.rs:33`). The caller runs the actual bulk
transfer through the xHCI driver, then feeds the 13-byte command status wrapper back for validation and
accounting (`src/server/handlers/accept_csw.rs:23`). The intended chain, from the capsule's own README,
is `driver.xhci0 -> driver.usb_msc0 -> block service -> filesystem capsules`
(`userland/capsule_driver_usb_msc/README.md:139`).

Everything the capsule keeps is process-local: the current endpoint binding table, a monotonic BOT tag
counter, and pass/fail/phase-error/residue counters (`src/state/types.rs:20`). It stores no product
strings, serial numbers, raw descriptors, SCSI payloads, or block data, and that memory drops on normal
userland teardown (`README.md:68`).

## Identity

Everything the kernel and the service registry need to name and reach the driver comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-usb-msc` | `Capsule.mk:6` |
| Capsule handle | `driver.usb_msc0` | `Capsule.mk:7`, `src/userspace/capsule_driver_usb_msc/spawn.rs:31` |
| Domain | `systems.nonos` | `Capsule.mk:8` |
| Namespace | `systems.nonos.driver.usb_msc0` | `Capsule.mk:12` |
| Service endpoint | `service:4224:driver.usb_msc0` | `Capsule.mk:13`, `spawn.rs:32` |
| Reply endpoint | `reply:4225:endpoint.4294967315` | `Capsule.mk:14`, `spawn.rs:33`, `spawn.rs:34` |
| Binary name | `driver_usb_msc` | `Capsule.mk:10` |
| Kernel feature | `nonos-capsule-driver-usb-msc` | `Capsule.mk:11` |
| Capability mask | `0x19` | `Capsule.mk:18` |
| Kernel mirror | `src/userspace/capsule_driver_usb_msc` | `src/userspace/mod.rs:38` |

The reply inbox string `endpoint.4294967315` is the decimal spelling of `0x1_0000_0013`; the kernel spawn
record stores it as the `REPLY_INBOX` name paired with reply port `4225`
(`src/userspace/capsule_driver_usb_msc/spawn.rs:33`, `spawn.rs:34`).

The mask `0x19` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x01  CoreExec   bit()  1   types.rs:56
  0x08  IPC        bit()  8   types.rs:59
  0x10  Memory     bit() 16   types.rs:60
  ----
  0x19  = 1 + 8 + 16
```

The kernel spawn path requests exactly those three capabilities and no others: `Capability::CoreExec.bit()
| Capability::IPC.bit() | Capability::Memory.bit()` (`src/userspace/capsule_driver_usb_msc/spawn.rs:51`).
There is no `DeviceEnum` (0x8000), no `Driver` (0x10000), no `Mmio` (0x20000), no `Irq` (0x40000), and no
`Dma` (0x80000) (`src/capabilities/types.rs:71`, `:72`, `:73`, `:74`, `:75`). There is also no `Debug`
bit: the `Capsule.mk` comment says it is deliberately absent so bulk-transfer payloads never reach the
serial surface, and the spawn spec passes an empty `debug_tag` (`Capsule.mk:17`, `spawn.rs:54`). That
empty hardware and driver surface is the whole basis of the security analysis below: this is a class
capsule that lives entirely above xHCI and speaks IPC only.

## Operation reference

Requests carry the shared 20-byte driver header (`NUMS`, version 1) followed by an operation-specific
body; the parser rejects anything whose declared payload length does not match the buffer exactly
(`src/protocol/header.rs:17`, `src/protocol/decode.rs:19`). The dispatcher matches on the 16-bit opcode
(`src/protocol/ops.rs:17`, `src/server/dispatch.rs:22`). Every reply begins with the same header echoed
back, then a 4-byte little-endian signed status word, then any op-specific payload
(`src/server/respond.rs:21`, `src/protocol/encode.rs:19`, `:29`).
Status `0` means success; a negative status is one of the errno constants in
`src/protocol/errno.rs`.

The eight opcodes, each cited to its handler:

| Op | Opcode | Request body | Reply payload after status | Handler |
|---|---|---|---|---|
| `OP_HEALTHCHECK` | `0x0001` | empty | none (status `0`) | `src/server/handlers/health.rs:20` |
| `OP_PROBE_CONFIG` | `0x0002` | raw USB configuration descriptor | `count_le32` then 8-byte binding records | `src/server/handlers/probe_config.rs:22` |
| `OP_BUILD_INQUIRY` | `0x0003` | empty | 31-byte BOT CBW for SCSI INQUIRY | `src/server/handlers/build_inquiry.rs:23` |
| `OP_BUILD_READ_CAPACITY10` | `0x0004` | empty | 31-byte BOT CBW for READ CAPACITY(10) | `src/server/handlers/build_capacity.rs:23` |
| `OP_BUILD_READ10` | `0x0005` | `lba_le32, blocks_le16` (6 bytes) | 31-byte BOT CBW for READ(10) | `src/server/handlers/build_read.rs:23` |
| `OP_BUILD_WRITE10` | `0x0006` | `lba_le32, blocks_le16` (6 bytes) | 31-byte BOT CBW for WRITE(10) | `src/server/handlers/build_write.rs:23` |
| `OP_ACCEPT_CSW` | `0x0007` | 13-byte BOT CSW | none; status carries the CSW status byte | `src/server/handlers/accept_csw.rs:22` |
| `OP_GET_STATE` | `0x0008` | empty | 48-byte counter snapshot | `src/server/handlers/get_state.rs:21` |

### Errors

The error codes are Linux-like negative errnos (`src/protocol/errno.rs:17`):

```
  E_INVAL     -22  malformed header, body, descriptor, or CSW signature
  E_BAD_OP    -38  unknown opcode with an empty body
  E_NO_MSC    -61  valid descriptor with no SCSI-transparent BOT interface
  E_OVERFLOW  -75  transfer block count above the bound
  E_PHASE     -84  CSW status byte out of range
```

Dispatch has one nuance worth stating plainly. The opcodes that must be empty (`OP_HEALTHCHECK`,
`OP_BUILD_INQUIRY`, `OP_BUILD_READ_CAPACITY10`, `OP_GET_STATE`) are guarded with a `body.is_empty()` arm
(`src/server/dispatch.rs:23`, `:25`, `:28`, `:34`). An unknown opcode with an empty body falls to the
`E_BAD_OP` arm, but an unknown opcode carrying a non-empty body falls to a final catch-all that answers
`E_INVAL`, not `E_BAD_OP` (`src/server/dispatch.rs:35`, `:38`). So "unknown op" is reported as `E_BAD_OP`
only when the body happens to be empty; otherwise the malformed-input answer wins.

### OP_PROBE_CONFIG detail

The body is a raw USB configuration descriptor. The parser requires at least 9 bytes with a configuration
descriptor type in `raw[1]`, reads the `wTotalLength` field, and refuses a total that is under 9 bytes or
longer than the buffer (`src/descriptors/parse.rs:22`). It then walks fixed-length records, rejecting any
record whose length is under 2 or that runs past the total (`src/descriptors/parse.rs:33`). Interface
records set the current candidate only when the class triple matches mass storage / SCSI-transparent /
Bulk-Only Transport; a non-matching interface clears the candidate (`src/descriptors/visitor.rs:29`).
Bulk endpoint records fill in the bulk-in or bulk-out address and its max packet size, and a binding is
emitted once both directions are present, up to `MAX_BINDINGS` (8) (`src/descriptors/visitor.rs:40`,
`src/protocol/limits.rs:19`). A descriptor with zero bindings answers `E_NO_MSC`; a malformed one answers
`E_INVAL` without mutating state (`src/descriptors/parse.rs:41`, `src/server/handlers/probe_config.rs:29`).
The reply is a 32-bit binding count followed by 8-byte records of
`interface, bulk_in, bulk_out, pad, max_packet_in_le16, max_packet_out_le16`
(`src/descriptors/encode.rs:19`).

### The CBW builders

INQUIRY, READ CAPACITY(10), READ(10), and WRITE(10) all build a `CommandBlockWrapper` with a fresh tag,
the expected data-transfer length, the direction flag, LUN 0, and a SCSI CDB, then serialize the 31-byte
BOT wrapper into the reply (`src/server/handlers/build_inquiry.rs:24`, `src/bot/cbw.rs:33`). INQUIRY and
READ CAPACITY(10) take no body and are IN transfers of 36 and 8 bytes respectively
(`src/server/handlers/build_inquiry.rs:28`, `src/server/handlers/build_capacity.rs:28`). READ(10) is an
IN transfer and WRITE(10) is an OUT transfer, each with a `data_len` of `blocks * 512`
(`src/server/handlers/build_read.rs:29`, `src/server/handlers/build_write.rs:29`,
`src/protocol/limits.rs:22`). Both read and write validate their 6-byte body through the same guard: the
body must be exactly 6 bytes, the block count must be non-zero, and the block count must not exceed
`MAX_TRANSFER_BLOCKS` (128), or the reply is `E_INVAL` or `E_OVERFLOW`
(`src/scsi/validate.rs:19`, `src/protocol/limits.rs:23`).

Note the endianness split, because it is a real wire detail and easy to get wrong. The IPC request body
carries the LBA and block count little-endian (`src/scsi/validate.rs:23`), but the SCSI CDB inside the
CBW carries them big-endian, as SCSI requires (`src/scsi/cdb.rs:41`). The capsule does that byte swap for
the caller.

### OP_ACCEPT_CSW detail

The body must be exactly 13 bytes with the CSW signature `USBS` (`0x53425355`); a wrong length or
signature answers `E_INVAL`, and a status byte above 2 answers `E_PHASE`
(`src/bot/csw.rs:28`, `:33`, `:39`). On a good parse the capsule folds the CSW into its counters and
returns the CSW's own status byte as the reply status (`src/server/handlers/accept_csw.rs:26`). A tag
that does not match the last issued tag bumps the phase-error counter; status `0` bumps `csw_ok`, status
`1` bumps `csw_failed`, and anything else bumps phase errors; the residue is summed
(`src/state/ops.rs:36`). This is the hook a recovery layer would read to decide the transport needs a
reset.

### OP_GET_STATE detail

Returns a 48-byte snapshot: `probes`, `csw_ok`, `csw_failed`, `phase_errors` as u64s, then
`binding_count` as u32, then `residue_bytes` as u64, then `last_tag` as u32, all little-endian
(`src/state/snapshot.rs:20`).

## Architecture and bring-up

### The Bulk-Only Transport

BOT wraps a SCSI command in a 31-byte Command Block Wrapper, transfers the data over a bulk endpoint,
then reads a 13-byte Command Status Wrapper. The capsule owns both wrappers.

The CBW it emits is the standard layout: signature `USBC` (`0x43425355`), the 4-byte tag, the expected
data-transfer length, a direction flag byte (`0x80` for IN, `0x00` for OUT), the LUN, the CDB length, and
16 bytes of CDB (`src/bot/cbw.rs:19`, `:20`, `:21`, `:33`). The CSW it validates is signature `USBS`, the
echoed tag, the residue, and a one-byte status (`src/bot/csw.rs:19`, `:28`).

### The SCSI commands

Four CDBs are built, each in `src/scsi/cdb.rs`:

- INQUIRY: opcode `0x12`, allocation length 36, CDB length 6 (`src/scsi/cdb.rs:17`).
- READ CAPACITY(10): opcode `0x25`, CDB length 10 (`src/scsi/cdb.rs:24`).
- READ(10): opcode `0x28`, big-endian LBA and transfer length, CDB length 10 (`src/scsi/cdb.rs:30`,
  `:38`).
- WRITE(10): opcode `0x2A`, same block-CDB shape, CDB length 10 (`src/scsi/cdb.rs:34`, `:38`).

That is the full SCSI vocabulary the capsule speaks today. There is no MODE SENSE, no REQUEST SENSE, no
TEST UNIT READY, and no sense-data decoding.

### How it sits on xHCI

The capsule is a pure consumer and producer of bytes. It never opens the controller. The README states
the boundary directly: PCI ownership, MMIO, IRQ routing, DMA, xHCI rings, endpoint configuration, and
bulk-transfer scheduling all stay in `driver.xhci0`; filesystems, partitioning, caching, and encryption
stay above the block layer (`README.md:20`). The runtime split of authority is the same: `driver.usb_msc0`
owns class-local state only, `driver.xhci0` owns device slots, endpoint contexts, transfer rings, and
bulk scheduling, and storage capsules own block-device registration and mount policy (`README.md:122`).

The xHCI transport is the sibling capsule this one depends on. Its broker-backed hardware authority (the
device claim, MMIO windows, IRQ binding, and DMA rings) is documented under the hardware broker:
[claim.md](../../subsystems/hardware-broker/claim.md), [mmio.md](../../subsystems/hardware-broker/mmio.md),
[dma.md](../../subsystems/hardware-broker/dma.md), and [irq.md](../../subsystems/hardware-broker/irq.md).
This MSC capsule holds none of that; it is the layer above.

### Bring-up sequence

1. The kernel spawns the capsule at boot through the USB spawn plan, gated on the
   `nonos-capsule-driver-usb-msc` feature; the plan runs `spawn_xhci` then `spawn_usb_hid` then
   `spawn_usb_msc` (`src/userspace/init/spawn_plan/drivers_usb.rs:17`, `:51`). The spawn verifies the
   embedded ELF, ID cert, manifest, and attestation trailer against the baked trust anchor, requests the
   `0x19` capability set, registers `driver.usb_msc0` on port 4224, and on success logs
   `[DRIVER-USB-MSC] capsule spawned` (`src/userspace/capsule_driver_usb_msc/spawn.rs:37`,
   `src/userspace/init/spawn_plan/drivers_usb.rs:54`, `src/userspace/init/capsule_boot/run.rs:29`).
2. The capsule's `_start` initializes the heap and enters the server loop
   (`src/main.rs:32`, `src/server/runner.rs:27`).
3. A storage caller sends `OP_PROBE_CONFIG` with the device's configuration descriptor; a successful
   probe replaces the endpoint snapshot (`src/state/ops.rs:23`).
4. The caller requests BOT wrappers, runs the bulk transfers through xHCI, and returns each CSW with
   `OP_ACCEPT_CSW`. The capsule never runs a transfer itself.

## Protocol and IPC

The capsule exposes exactly the eight opcodes above on the `driver.usb_msc0` service (port 4224) and
replies on the reply inbox (port 4225) (`src/userspace/capsule_driver_usb_msc/spawn.rs:32`, `:34`). It
makes no outbound IPC calls of its own. It is a pure server: receive, decide, reply.

The wire is the `NUMS` envelope. The 20-byte header is magic `0x4E554D53` ("NUMS"), a `u16` version of 1,
the `u16` opcode, a `u16` flags field, a two-byte pad, a `u32` request id, and a `u32` payload length
(`src/protocol/header.rs:17`, `src/protocol/decode.rs:23`). The parser is strict: it rejects a short
buffer, a wrong magic or version, and any message whose declared payload length plus the header does not
equal the received length exactly (`src/protocol/decode.rs:20`, `:25`, `:33`). The response header echoes
the request's op, flags, and request id, zeroes the pad, and writes the reply payload length
(`src/protocol/encode.rs:19`).

The receive/reply mechanics use two libc primitives and nothing else. The runner blocks on
`mk_ipc_recv_from` against inbox 0, captures the sender pid, drops any message with a non-positive length
or a zero sender, parses, and dispatches (`src/server/runner.rs:37`, `:38`, `:41`). Handlers reply with
`mk_ipc_reply` to the captured sender pid (`src/server/respond.rs:24`, `:31`). The buffers are sized once
at startup to `HDR_LEN + IPC_PAYLOAD_MAX` (20 + 1024 bytes) and reused for every request
(`src/server/runner.rs:28`, `src/protocol/limits.rs:17`). There is no `mk_pio_*`, `mk_dma_*`, `mk_mmio_*`,
`mk_irq_*`, or `mk_device_*` call anywhere in the capsule; the static CI gate enforces exactly that
(`nonos-ci/run-static-checks.sh:1403`).

## Security analysis

This capsule has no hardware authority. Its mask is `0x19`: CoreExec, IPC, and Memory, and nothing else
(`Capsule.mk:18`, `src/userspace/capsule_driver_usb_msc/spawn.rs:51`). It holds no `DeviceEnum`, `Driver`,
`Mmio`, `Irq`, or `Dma` capability, so it cannot enumerate a USB controller, claim a PCI device, map a
register window, bind an interrupt, or allocate a DMA buffer. The hardware broker never grants it a
device claim, an MMIO region, an IRQ slot, or a DMA ring, because it never asks for one; all of that
authority belongs to `driver.xhci0`, and is described under the hardware broker docs linked above. The
no-IOMMU caveat that applies to real DMA-capable drivers on this platform does not apply here at all,
precisely because this capsule holds no DMA authority and touches no physical memory of a device; the DMA
exposure lives one layer down, in the xHCI driver.

What it can do is bounded to descriptor classification, BOT/SCSI framing, and status accounting. It
receives descriptor bytes and CSW bytes over IPC and returns class decisions and command wrappers
(`README.md:39`). A caller cannot use it to reach memory it does not own: the capsule reads only the
request body it was handed and writes only into its own reply buffer, sized once at startup
(`src/server/runner.rs:28`). Every length is checked before use. The descriptor walk refuses records that
run past the declared total (`src/descriptors/parse.rs:35`); the block-request guard refuses a body that
is not exactly 6 bytes, a zero count, and a count over 128 (`src/scsi/validate.rs:20`, `:25`, `:28`); the
CSW parser refuses a wrong length, a bad signature, and a status byte over 2 (`src/bot/csw.rs:29`, `:33`,
`:39`). Malformed input fails closed without mutating the endpoint snapshot or the counters.

Isolation from other capsules is the kernel's, not the driver's. It is a CPL 3 user binary that speaks
IPC and holds three capability bits; it is verified, its cert and manifest and attestation checked, and
enrolled at spawn like every other capsule (`src/userspace/capsule_driver_usb_msc/spawn.rs:38`, `:56`).
The static gate additionally forbids the source from importing any kernel driver, memory, paging, phys,
or hardware path, and asserts the `0x19` boundary, so the isolation is enforced at build time as well as
at spawn (`nonos-ci/run-static-checks.sh:1394`, `:1412`).

## Implemented versus stub

Implemented and exercised on the wire today (`README.md:90`):

- USB configuration descriptor walk with strict record bounds (`src/descriptors/parse.rs:22`).
- SCSI-transparent BOT interface detection by the class/subclass/protocol triple
  (`src/descriptors/visitor.rs:34`).
- Bulk-in and bulk-out endpoint extraction, up to 8 bindings (`src/descriptors/visitor.rs:47`).
- BOT command block wrapper construction with a monotonic tag (`src/bot/cbw.rs:33`, `src/state/ops.rs:29`).
- BOT command status wrapper validation and counter accounting (`src/bot/csw.rs:28`, `src/state/ops.rs:36`).
- INQUIRY, READ CAPACITY(10), READ(10), and WRITE(10) CDB construction (`src/scsi/cdb.rs:17`).
- Bounded transfer-length validation (`src/scsi/validate.rs:28`).
- Counter snapshot for introspection (`src/state/snapshot.rs:20`).
- Kernel-spawnable, signed capsule metadata and a stable endpoint contract
  (`src/userspace/capsule_driver_usb_msc/spawn.rs:37`).

Explicitly not implemented in this slice, stated as non-goals by the capsule itself (`README.md:168`):

- xHCI bulk-transfer scheduling. The capsule builds a CBW but never sends it; there is no transfer call.
- USB reset and error recovery. `OP_ACCEPT_CSW` counts a phase error but does not act on it.
- Multi-LUN handling. Every CBW is built with LUN 0 hard-coded (`src/server/handlers/build_inquiry.rs:29`).
- UASP, SCSI sense decoding, filesystem mounting, writeback caching, partition parsing, encryption, and
  block-device publication.

So READ(10) and WRITE(10) are not stubbed in the sense of returning a placeholder: they build a correct,
bounded CBW. But they are not end-to-end I/O either, because this capsule never performs the transfer.
The read and write paths stop at "here is the command wrapper to send"; running it and returning data is
the caller's job through xHCI, and that runtime chain is a later slice (`README.md:137`, `README.md:150`).

## How to contribute

The source lives at `userland/capsule_driver_usb_msc/`. The layout is one concern per module tree:
`protocol/` is the wire, `descriptors/` is the USB parse, `bot/` is the CBW/CSW, `scsi/` is the CDBs,
`state/` is the process-local counters and bindings, and `server/` is the request loop and handlers.

To add an operation:

1. Add the opcode constant in `src/protocol/ops.rs:17` and re-export it from `src/protocol/mod.rs:31`.
2. Write the handler as one file under `src/server/handlers/`, following the shape of the existing ones:
   take `state`, `sender_pid`, `req`, optionally `body`, and `tx`, do the work, and reply through
   `respond::status` or `respond::payload` (`src/server/respond.rs:21`, `:27`). Register it in
   `src/server/handlers/mod.rs:17`.
3. Wire the opcode into the match in `src/server/dispatch.rs:22`. If the op takes no body, add the
   `body.is_empty()` guard the way the empty-body ops do (`src/server/dispatch.rs:23`).
4. Keep the capability boundary. Do not import any kernel driver, memory, or hardware path, and do not
   call a `mk_pio_*`, `mk_dma_*`, `mk_mmio_*`, `mk_irq_*`, or `mk_device_*` syscall; the static gate fails
   the build otherwise (`nonos-ci/run-static-checks.sh:1394`, `:1403`).

To build and sign the capsule, use the per-slug make targets generated by `nonos-mk/capsule.mk` for the
`driver-usb-msc` slug (`nonos-mk/capsule.mk:158`, included through
`userland/capsule_driver_usb_msc/Capsule.mk:20`):

```
  make nonos-mk-driver-usb-msc              build the capsule ELF
  make nonos-mk-driver-usb-msc-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-driver-usb-msc-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-driver-usb-msc-keys   check the per-capsule signing keys exist
```

These are declared for the slug at `Makefile:31` (the `.PHONY` line that lists
`nonos-mk-driver-usb-msc nonos-mk-driver-usb-msc-sign nonos-mk-check-driver-usb-msc-keys`) and generated
by the macro at `nonos-mk/capsule.mk:158`. For a kernel image that includes the driver,
`make nonos-mk-driver-usb-msc-prod` builds the profile with `KERNEL_FEATURES :=
microkernel-driver-usb-msc`, pulling in the proof-io, xHCI, and MSC artifacts as dependencies
(`Makefile:985`, `Makefile:986`). The capsule's own README lists the same build entry
`make -B nonos-mk-driver-usb-msc` plus the static gate `bash nonos-ci/run-static-checks.sh`
(`README.md:177`, `README.md:180`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns an errno as a status word, never a panic; the release
profile is `panic = "abort"`, `Cargo.toml:25`); modular files, one unit per file, with `mod.rs` used only
for re-exports (`src/protocol/mod.rs`, `src/server/mod.rs`); and the AGPL header at the top of every
source file, matching the header on every existing module (`src/main.rs:1`).

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[DRIVER-USB-MSC] capsule spawned`, from the `[<tag>] <msg>` boot-log format with tag `DRIVER-USB-MSC`
and message `capsule spawned` (`src/userspace/init/spawn_plan/drivers_usb.rs:55`,
`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line means the
capsule never started; the error path prints an `[ERROR]` line with the specific spawn reason instead
(`src/userspace/init/capsule_boot/run.rs:32`, `src/sys/boot_log/output.rs:49`). Note the capsule holds no
`Debug` capability by design, so it emits no serial diagnostics of its own during operation; the marker
above is the only boot-time signal (`Capsule.mk:17`).

Failure modes and where to look:

- Spawn absent or `[ERROR]` at boot. Usually the feature is off (the binary is not embedded), or the
  signature, manifest, cert, or attestation failed. The spawn is gated behind
  `nonos-capsule-driver-usb-msc`; without it the plan compiles a no-op `spawn_usb_msc`
  (`src/userspace/init/spawn_plan/drivers_usb.rs:51`, `:62`). The specific error text comes from the spawn
  reason mapper (`src/userspace/init/capsule_boot/error.rs:21`).
- Probe answers `E_NO_MSC` (-61). The descriptor parsed but held no SCSI-transparent BOT interface with
  both bulk directions; check the class triple `0x08 / 0x06 / 0x50` and that both a bulk-in and a bulk-out
  endpoint are present (`src/descriptors/visitor.rs:34`, `:47`, `src/descriptors/parse.rs:41`).
- Probe answers `E_INVAL` (-22). The descriptor is malformed: too short, wrong type byte, a `wTotalLength`
  out of range, or a record length that runs past the total (`src/descriptors/parse.rs:23`, `:27`, `:35`).
  The endpoint snapshot is left untouched.
- A read or write answers `E_INVAL` or `E_OVERFLOW`. The 6-byte body guard failed: wrong length, zero
  block count, or a count above 128 (`src/scsi/validate.rs:20`, `:25`, `:28`).
- A CSW answers `E_INVAL` or `E_PHASE`. `E_INVAL` is a wrong length or a bad `USBS` signature; `E_PHASE`
  is a status byte above 2 (`src/bot/csw.rs:29`, `:33`, `:39`). A tag mismatch does not error; it silently
  bumps the phase-error counter, which you read back with `OP_GET_STATE`
  (`src/state/ops.rs:37`, `src/state/snapshot.rs:24`).
- An unknown opcode answers `E_BAD_OP` (-38) only if its body was empty; with a non-empty body it answers
  `E_INVAL` instead (`src/server/dispatch.rs:35`, `:38`).

`OP_GET_STATE` is the introspection probe: it returns the probe count, CSW pass/fail totals, phase-error
count, current binding count, summed residue, and last tag, so a caller can distinguish a transport that
never bound from one that bound but is failing its transfers (`src/state/snapshot.rs:20`).

## Source map

```
  src/main.rs                          _start -> heap_init -> server::run
  src/protocol/header.rs               NUMS magic, version, 20-byte header, Request
  src/protocol/decode.rs               strict request parse
  src/protocol/encode.rs               response header + status word
  src/protocol/ops.rs                  the eight opcodes
  src/protocol/errno.rs                E_INVAL / E_BAD_OP / E_NO_MSC / E_OVERFLOW / E_PHASE
  src/protocol/limits.rs               payload max, CBW/CSW lengths, block size, transfer bound
  src/descriptors/parse.rs             configuration descriptor walk with bounds
  src/descriptors/visitor.rs           interface class match + bulk endpoint binding
  src/descriptors/wire.rs              descriptor type and class constants
  src/descriptors/encode.rs            binding-count + 8-byte binding records
  src/bot/cbw.rs                       31-byte Command Block Wrapper writer
  src/bot/csw.rs                       13-byte Command Status Wrapper parser
  src/scsi/cdb.rs                      INQUIRY, READ CAPACITY(10), READ(10), WRITE(10) CDBs
  src/scsi/validate.rs                 6-byte block-request guard (LE body)
  src/state/types.rs                   bindings, tag counter, pass/fail/phase/residue counters
  src/state/ops.rs                     install_bindings, next_tag, accept_csw
  src/state/snapshot.rs                48-byte counter snapshot
  src/server/runner.rs                 the receive/parse/dispatch/reply loop
  src/server/dispatch.rs               opcode -> handler match
  src/server/respond.rs               status and payload reply helpers
  src/server/handlers/                 one file per op
  Capsule.mk                           slug, handle, ports, capability mask, feature
  src/userspace/capsule_driver_usb_msc/spawn.rs   the kernel-side verified spawn (0x19 caps)
  src/userspace/init/spawn_plan/drivers_usb.rs    the USB spawn plan (xhci, usb_hid, usb_msc)
  nonos-mk/capsule.mk                  the generated nonos-mk-driver-usb-msc[-sign|-verify] targets
  nonos-ci/run-static-checks.sh        the capability-boundary and MSC-path static gates
```

Every reference above is verified against those trees.

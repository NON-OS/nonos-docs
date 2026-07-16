# capsule_driver_hda (full reference)

`capsule_driver_hda` is the Intel High Definition Audio driver in the NONOS tree: a userland capsule
that owns one HDA PCI controller, brings it out of reset, and answers controller, codec, and
stream-layout queries over IPC. It is a controller and codec inventory service, not an audio pipeline.
This first slice does discovery, claim, register bring-up, codec-presence and vendor-id probing, and
GCAP-derived stream-descriptor layout; it does not play or capture audio. There is no CORB/RIRB verb
ring, no buffer descriptor list, and no stream DMA in this slice
(`userland/capsule_driver_hda/README.md:143`). It reaches hardware exclusively through the
[hardware broker](../../subsystems/hardware-broker/README.md), never through kernel driver code.

The kernel spawns it as part of the storage/driver fleet under service handle `driver.hda0` on service
port 4218 with a reply port on 4219, and its manifest capability mask is `0x78019`
(`userland/capsule_driver_hda/Capsule.mk:17`). The source is `userland/capsule_driver_hda/`.

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
the error (`userland/capsule_driver_hda/src/main.rs:37`). The two halves are clean: `setup::run` performs
every privileged bring-up step once and returns a `Driver` that owns the broker grants and the mapped
register window, and `server::run` is an endless request loop that never touches the broker again except
to poll and acknowledge the controller interrupt (`src/setup/sequence.rs:27`, `src/server/runner/run.rs:32`).

Setup discovers the HDA PCI function, claims it, enables bus mastering, maps the BAR0 register window,
binds the controller interrupt, brings the controller out of reset, reads the controller-global
registers, and probes each present codec for its vendor and device id
(`src/setup/sequence.rs:27`). Everything the driver returns is derived from those registers. The codec
probe uses the controller's immediate-command registers (`IC`/`IR`/`IRS`), not a CORB/RIRB ring, and it
sends exactly one verb, `Get Parameter(Vendor ID)`, per present codec
(`src/controller/immediate.rs:23`, `src/controller/codec_probe.rs:48`, `README.md:147`). The stream
layout is computed from `GCAP`, not read from live stream descriptors: the driver counts the input,
output, and bidirectional streams `GCAP` advertises and reports the standard descriptor offset each one
would occupy (`src/controller/stream_layout.rs:32`, `src/controller/streams.rs:17`). No sample ever moves.

## Identity

Everything the kernel and the service registry need to name and reach the driver comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-hda` | `Capsule.mk:7` |
| Service handle | `driver.hda0` | `Capsule.mk:8`, `src/hardware/hda_capsule/spawn.rs:32` |
| Namespace | `systems.nonos.driver.hda0` | `Capsule.mk:13` |
| Service endpoint | `service:4218:driver.hda0` | `Capsule.mk:14`, `spawn.rs:33` |
| Reply endpoint | `reply:4219:endpoint.4294967312` | `Capsule.mk:15`, `spawn.rs:34` |
| Reply inbox (kernel) | `endpoint.4294967312` (`0x1_0000_0010`) | `src/hardware/hda_capsule/client/transport.rs:25`, `src/protocol/endpoint.rs:18` |
| Capability mask | `0x78019` | `Capsule.mk:17` |
| Binary name | `driver_hda` | `Capsule.mk:11` |
| Kernel mirror | `src/hardware/hda_capsule` | `src/hardware/hda_capsule/mod.rs` |

The reply endpoint name `endpoint.4294967312` is the decimal spelling of the reply-inbox id the capsule
sends every response to. That id is `KERNEL_REPLY_ENDPOINT = 0x1_0000_0010` in the capsule
(`src/protocol/endpoint.rs:18`) and `REPLY_INBOX = "endpoint.4294967312"` on the kernel side
(`src/hardware/hda_capsule/client/transport.rs:25`); `0x1_0000_0010` is 4294967312, so the two agree.

The mask `0x78019` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x00001  CoreExec     bit()      1     types.rs:56
  0x00008  IPC          bit()      8     types.rs:59
  0x00010  Memory       bit()     16     types.rs:60
  0x08000  DeviceEnum   bit()  32768     types.rs:71
  0x10000  Driver       bit()  65536     types.rs:72
  0x20000  Mmio         bit() 131072     types.rs:73
  0x40000  Irq          bit() 262144     types.rs:74
  -------
  0x78019  = 1 + 8 + 16 + 32768 + 65536 + 131072 + 262144
```

Those are the capabilities an MMIO device driver with an interrupt needs and nothing more. `DeviceEnum`
is enumerate-only, `Driver` allows claim and release of one device, `Mmio` maps a device's register
window, and `Irq` binds its interrupt; the comment on the capability enum spells out that split
(`src/capabilities/types.rs:35`). Crucially, there is no `Dma` bit (0x80000): unlike the AHCI or NVMe
drivers, this slice never allocates a device-visible buffer, because it never programs a stream. There is
also no `Pio` bit (0x100000), no `FileSystem` bit (64), no `Network` bit (4), and no `Admin` (512) or
`Debug` (256). The driver cannot touch an I/O port, do DMA, read a file, open a socket, or reach raw
kernel memory.

Two identity notes worth recording, because they are places where the code and its neighbours can drift:

- The `Capsule.mk` comment on line 16 lists `IPC|Memory|Driver|DeviceEnum|Mmio|Irq = 0x78019`, but that
  six-name set sums to `0x78018`. The extra `0x1` in the real mask is `CoreExec`, which every runnable
  capsule needs and which the comment simply omits. The numeric value `0x78019` on line 17 is the
  authority signed into the manifest, and it is the one decomposed above.
- The kernel-side spawn record requests only the six hardware and IPC bits and not `CoreExec` explicitly
  (`src/hardware/hda_capsule/spawn.rs:51`). The signed manifest mask `0x78019` (`Capsule.mk:17`) is the
  ceiling the trust anchor enforces; the spawn `requested_caps` is a request bounded by that manifest,
  and `CoreExec` is granted to the capsule as a matter of being an executable process.

## Operation reference

The service accepts one request at a time on its endpoint, decodes the 20-byte header, and dispatches on
the operation id (`src/server/runner/run.rs:54`). Every reply reuses the same header shape and begins
with a 4-byte little-endian status word; `E_OK` is success and `E_INVAL` is the only error the runtime
returns (`src/protocol/errno.rs:17`). The opcodes are defined in `src/protocol/ops.rs`.

Errno values (`src/protocol/errno.rs:17`):

```
  E_OK       0     success
  E_INVAL  -22     bad magic/version, an unknown opcode, or any op carrying a payload
```

| Op | Opcode | Request payload | Reply payload | Errors | Handler |
|---|---|---|---|---|---|
| `OP_HEALTHCHECK` | 1 | none | status word only | `E_INVAL` if payload present | `ops.rs:17`, `handlers/health.rs:20` |
| `OP_CONTROLLER_INFO` | 2 | none | status + 28-byte register record | `E_INVAL` if payload present | `ops.rs:18`, `handlers/controller_info.rs:26` |
| `OP_CODEC_MASK` | 3 | none | status + 8-byte mask payload | `E_INVAL` if payload present | `ops.rs:19`, `handlers/codec_mask.rs:26` |
| `OP_STREAM_LAYOUT` | 4 | none | status + 4-byte count + 8-byte stream entries | `E_INVAL` if payload present | `ops.rs:20`, `handlers/stream_layout.rs:26` |
| `OP_CODEC_LIST` | 5 | none | status + 4-byte count + 8-byte codec entries | `E_INVAL` if payload present | `ops.rs:21`, `handlers/codec_list.rs:25` |

Request and reply header (20 bytes, `src/protocol/header.rs:17`, `src/protocol/decode.rs:19`,
`src/protocol/encode.rs:19`):

```
  u32 magic      = 0x4e48_4441  "NHDA"   header.rs:17
  u16 version    = 1                     header.rs:18
  u16 op                                 decode.rs:30
  u16 flags                              decode.rs:31
  u16 reserved                           encode.rs:24
  u32 request_id                         decode.rs:32
  u32 payload_len                        decode.rs:33
```

A request whose first four bytes are not `NHDA` or whose version is not 1 is dropped with `E_INVAL` and
no register is touched (`src/protocol/decode.rs:23`, `src/server/error.rs:29`). Every op in this slice is
fixed-size and takes no payload, so the runner rejects any request that declares a non-zero
`payload_len` with `E_INVAL` before dispatch (`src/server/runner/run.rs:50`).

The reply bodies are all built directly by the handlers. `OP_CONTROLLER_INFO` re-reads the HDA global
registers live at request time and returns a 28-byte record: `GCAP`, `VMIN`, `VMAJ`, `OUTPAY`, `INPAY`,
`GCTL`, `STATESTS`, `GSTS`, `INTCTL`, `INTSTS`, then the four GCAP-derived counts (input streams, output
streams, bidirectional streams, 64-bit-address flag) (`handlers/controller_info.rs:32`,
`src/controller/info.rs:40`). `OP_CODEC_MASK` returns the low 15 bits of `STATESTS` as a codec-presence
mask, two reserved bytes, and the popcount of that mask as a 4-byte set count
(`handlers/codec_mask.rs:28`). `OP_STREAM_LAYOUT` returns a 4-byte entry count followed by one 8-byte
entry per stream: `u8 kind` (1 input, 2 output, 3 bidirectional), `u8 local_index`, `u16 global_index`,
`u32 stream_descriptor_offset` (`handlers/stream_layout.rs:35`, `src/controller/streams.rs:20`). The
offset is the standard `0x80 + global_index * 0x20` descriptor position, computed rather than read from a
live descriptor (`src/controller/stream_layout.rs:48`). `OP_CODEC_LIST` returns a 4-byte count of present
codecs followed by one 8-byte entry per present codec: `u8 codec_address`, `u8 probe_ok`, `u16
vendor_id`, `u16 device_id`, `u16 reserved` (`handlers/codec_list.rs:33`).

## Architecture and bring-up

The whole privileged path runs once in `setup::run` and unwinds prior broker grants on any failure
(`src/setup/sequence.rs:27`). The steps, in order:

1. Discover. `find_hda` asks the broker for the audio-class device list and returns the first record that
   is a PCI multimedia controller with subclass HDA (`0x04`/`0x03`), with BAR0 present as MMIO of at
   least 4 KiB, a non-zero interrupt pin, and a routed interrupt line
   (`src/discover.rs:32`, `:50`, `src/constants/pci.rs:17`).
2. Claim. `mk_device_claim` binds the controller to this process and returns a claim epoch; every later
   broker call carries that epoch (`src/setup/claim.rs:21`).
3. Bus master. `enable_bus_master` writes the PCI command register to set the bus-master bit through
   `mk_pci_config_write` on the claimed function (`src/setup/pci.rs:21`,
   `userland/libc/src/broker/pci.rs:26`). Failure here releases the device before returning
   (`src/setup/sequence.rs:30`).
4. Map BAR0. `mmio::map` maps BAR0, the HDA register window, into the capsule's address space and returns
   a user VA and a grant id; failure releases the device (`src/setup/mmio.rs:22`,
   `src/constants/pci.rs:18`). A thin `Regs` wrapper does volatile 8/16/32-bit reads and 8/32-bit writes
   at `base + offset` (`src/regs/mmio.rs:29`).
5. Bind IRQ. `irq::bind` binds the controller's interrupt line and returns a grant id; failure unmaps
   BAR0 and releases the device (`src/setup/irq.rs:22`). The interrupt is polled and acked at runtime but
   is not required for any query, which are all synchronous register reads.
6. Leave reset. `leave_reset` reads `GCTL`, sets `GCTL.CRST` (bit 0) if it is clear, and spins until the
   controller reports `CRST` set, meaning it has come out of reset and is running; a bounded spin that
   never clears returns `ControllerResetTimeout` (`src/controller/reset.rs:23`,
   `src/constants/regs.rs:22`, `:31`). Note that on HDA `CRST = 1` is the run state and `CRST = 0` is the
   reset state, so this step drives the bit high and waits for it, the opposite polarity from the AHCI
   `GHC.HR` self-clearing reset.
7. Read controller info and reject an unusable controller. `ControllerInfo::read` snapshots the global
   registers and derives the stream counts and address width from `GCAP`
   (`src/controller/info.rs:40`, `src/controller/streams.rs:17`). If the major version or `GCAP` reads
   back as zero, setup fails with `UnsupportedController` (`src/setup/sequence.rs:41`).
8. Probe codecs. `probe` walks the 15 codec slots, treats a slot as present when its `STATESTS` bit is
   set, and for each present slot sends `Get Parameter(Vendor ID)` through the immediate-command
   interface, recording the vendor id (high 16 bits) and device id (low 16 bits), or marking the probe as
   failed if the command times out (`src/controller/codec_probe.rs:32`, `:47`).

### The immediate-command codec interface

This slice reads codec identity through the controller's immediate-command registers, not a CORB/RIRB
ring. `get_parameter` composes a 32-bit verb from the codec address, node id, the 12-bit verb
`Get Parameter` (`0x0f00`), and an 8-bit parameter, then sends it: it waits for `IRS.BUSY` to clear,
writes `IRS.VALID`, writes the verb to `IC`, sets `IRS.BUSY`, and spins until `BUSY` clears with `VALID`
set before reading the response from `IR`; either wait can time out (`src/controller/immediate.rs:23`,
`:37`, `:49`, `:62`, `src/constants/regs.rs:27`, `:32`). The only verb this slice ever issues is
`Get Parameter(Vendor ID)` for inventory (`README.md:147`). CORB/RIRB verb transport is a named future
target, not present here (`README.md:124`, `:143`).

### The stream layout derivation

The stream layout is a computed projection of `GCAP`, not a read of live stream descriptors. `GCAP` bits
8..11 give the input-stream count, bits 12..15 the output-stream count, bits 3..7 the bidirectional
count, and bit 0 the 64-bit-address capability (`src/controller/streams.rs:17`). `layout` appends the
input, then output, then bidirectional descriptors in that order, assigning each a running global index
and the standard descriptor offset `0x80 + global_index * 0x20`, capped at 64 descriptors
(`src/controller/stream_layout.rs:32`, `:41`). No stream register is read and no stream is programmed;
the offsets tell a client where the descriptors would live once playback and capture are implemented.

### There is no DMA or playback path

This is the honest boundary of the slice. The driver holds no `Dma` capability and calls no DMA broker
syscall, so it allocates no command ring, no buffer descriptor list, and no sample buffer. The README is
explicit that CORB/RIRB, stream descriptor programming, BDL, PCM playback, PCM capture, mixer, jack
policy, and volume policy are all absent, and that the only codec verb path is the immediate-command
`Get Parameter(Vendor ID)` used for inventory (`README.md:143`, `:147`). Audio playback and capture come
later, after CORB/RIRB and stream DMA are real (`README.md:11`).

### IRQ handling

The server loop polls the controller interrupt each iteration through `mk_irq_poll` on the IRQ grant id;
when the sequence number advances it acknowledges with `mk_irq_ack` and records the new sequence
(`src/server/runner/poll_irq.rs:21`). This keeps the interrupt drained but gates nothing: every op is a
synchronous register read, so no request waits on an interrupt. The interrupt is bound so the controller
is owned end to end and does not raise an unhandled line.

## Protocol and IPC

The driver is a server. It never calls another userland service; every outbound call it makes is a broker
syscall, and every inbound call is a query on its endpoint.

Inbound, service `driver.hda0` on `service:4218:driver.hda0`: the loop blocks on `mk_ipc_recv`, decodes
the `NHDA` header, dispatches one of the five ops above, and sends the reply to `KERNEL_REPLY_ENDPOINT`
with `mk_ipc_send` (`src/server/runner/run.rs:39`, `src/server/error.rs:26`).

Outbound broker syscalls, each through `nonos_libc` and each documented in the hardware broker subsystem:

```
  mk_device_list       enumerate audio-class devices          discover.rs:34                 (broker device list)
  mk_device_claim      claim the HDA function                  setup/claim.rs:22              claim.md
  mk_pci_config_write  set the PCI bus-master bit              setup/pci.rs:22                claim.md
  mk_mmio_map          map BAR0 into the capsule VA            setup/mmio.rs:24               mmio.md
  mk_mmio_unmap        unmap BAR0 on failure or teardown       setup/mmio.rs:26, handles/broker_handles_drop.rs:24   mmio.md
  mk_irq_bind          bind the controller interrupt           setup/irq.rs:24                irq.md
  mk_irq_poll          poll for interrupt events               server/runner/poll_irq.rs:23   irq.md
  mk_irq_ack           acknowledge a delivered interrupt       server/runner/poll_irq.rs:25   irq.md
  mk_irq_unbind        release the interrupt on teardown       handles/broker_handles_drop.rs:23   irq.md
  mk_device_release    release the device claim                setup/*, handles/broker_handles_drop.rs:25   claim.md
```

There is no `mk_dma_map` call anywhere in the capsule, which matches the missing `Dma` bit in the mask.
The [claim](../../subsystems/hardware-broker/claim.md), [mmio](../../subsystems/hardware-broker/mmio.md),
and [irq](../../subsystems/hardware-broker/irq.md) broker pages describe how each grant is validated,
bounded to the claim epoch, and revoked; the [dma](../../subsystems/hardware-broker/dma.md) page covers
the DMA path this slice does not yet use. All of the driver's broker grants are held in one owner that
frees them on drop: `BrokerHandles` releases the IRQ bind, the BAR0 mapping, and the device claim in that
order (`src/handles/broker_handles_drop.rs:22`).

## Security analysis

The driver holds real hardware authority over one HDA controller, and that authority is the point of the
design: audio hardware ownership is a signed, sandboxed, single-device userland capsule that can only
reach the controller through broker-mediated grants. Its mask `0x78019` grants `CoreExec`, `IPC`,
`Memory`, `DeviceEnum`, `Driver`, `Mmio`, and `Irq` and nothing else (`Capsule.mk:17`). There is no `Dma`
bit, so the driver cannot allocate a device-visible buffer or start a bus-mastering stream; no `Pio` bit,
so it cannot fall back to raw port I/O; no `FileSystem` bit, so it cannot read or write a file; no
`Network` bit; and no `Admin` or `Debug`.

What the broker grants and bounds:

- The device claim binds exactly one HDA function to this process and issues an epoch that every later
  MMIO and IRQ call must carry (`src/setup/claim.rs:22`). A capsule cannot act on a device it did not
  claim, and the kernel revokes the claim and every grant behind it on exit.
- `mk_mmio_map` maps only BAR0, the HDA register window, and only for this claim; the driver never sees
  any other register space (`src/setup/mmio.rs:24`). Reads and writes go through a volatile wrapper at
  offsets inside that window (`src/regs/mmio.rs:29`).
- The bus-master bit is set through `mk_pci_config_write` against the claimed function, not by poking
  config space directly (`src/setup/pci.rs:22`). It is enabled ahead of the future stream-DMA work; in
  this slice the controller does no DMA because the driver programs no stream.

The no-IOMMU caveat is the honest limit for any bus-mastering device, and it is why this slice is
deliberately narrow. The bus-master bit is set, but the driver never hands the controller a DMA address,
never programs a buffer descriptor list, and never starts a stream, so it initiates no bus-master
transfer today. When stream DMA is added, the same caveat that applies to every NONOS bus-mastering
driver will apply here: on a platform where the kernel has not engaged an IOMMU, the hardware itself does
not confine a bus-mastering controller to the buffers it was given, so the enforcement is the driver's
correctness plus the broker's grant discipline, not a hardware boundary. That caveat is spelled out on
the broker [dma](../../subsystems/hardware-broker/dma.md) page, and the capsule keeps `Dma` out of its
mask until that path is real (`README.md:52`).

Isolation from other capsules is the kernel's, not the driver's: it is a CPL 3 user binary, verified and
enrolled at spawn like every other capsule, that speaks only IPC and broker syscalls. A bug in header
parsing or in a handler cannot escalate past the seven capabilities in the mask, because the driver never
held more than the right to claim one device and read its registers. Requests are strictly validated
before any register is touched: a bad magic or version is dropped, and any op carrying a payload is
rejected with `E_INVAL`, so the entire input surface is a 20-byte header with an opcode and no body
(`src/server/runner/run.rs:43`, `:50`). Because no op ever accepts caller data and no op programs the
device, the request path cannot drive the controller into an attacker-chosen state; it can only read what
the controller reports.

## How to contribute

The source lives at `userland/capsule_driver_hda/`. The tree is one unit per file: discovery is
`src/discover.rs`, the one-shot bring-up is under `src/setup/`, the controller reads, reset, codec probe,
immediate-command interface, and stream layout are under `src/controller/`, the register offsets and PCI
constants are under `src/constants/`, the volatile register wrapper is under `src/regs/`, the wire format
is under `src/protocol/`, the request loop and per-op handlers are under `src/server/`, and the
broker-grant owner is under `src/handles/`.

To add or change an operation:

1. Add the opcode to `src/protocol/ops.rs` and any fixed sizes to `src/protocol/limits.rs`; the reply
   transmit buffer is sized from those limits at compile time (`src/server/runner/max_tx_body.rs:19`).
2. Write the handler as one file under `src/server/handlers/`, exposing a `pub fn handle(...)` that
   builds the reply with `encode_response_header` and `write_status` and sends it with `mk_ipc_send`, or
   returns an errno through `reply_with_status` (`src/server/error.rs:23`). Re-export it from
   `src/server/handlers/mod.rs`.
3. Wire it into the match in `src/server/runner/run.rs:54`; the payload-length guard at `:50` already
   rejects any op that carries a body, so keep new query ops payload-free or relax that guard
   deliberately.
4. If it needs a new register or codec verb, add the offset to `src/constants/regs.rs` and the access
   under `src/controller/`, keeping raw MMIO behind the `Regs` wrapper (`src/regs/mmio.rs:29`).

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_driver_hda/Capsule.mk:19`):

```
  make nonos-mk-driver-hda             build the capsule ELF
  make nonos-mk-driver-hda-sign        produce the id cert, manifest, and attestation trailer
  make nonos-mk-driver-hda-verify      verify the signed artifacts against the trust anchor
  make nonos-mk-check-driver-hda-keys  check the per-capsule signing keys exist
```

For a running kernel that embeds and spawns the driver, `make nonos-mk-driver-hda-prod` builds the kernel
image with the `microkernel-driver-hda` feature so the driver fleet spawns `driver_hda` at boot
(`Makefile:1010`). The README documents the build and static-gate entry points, including
`make -B nonos-mk-driver-hda` and `bash nonos-ci/run-static-checks.sh` (`README.md:152`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every fallible path returns an `HdaError` or an errno, and the release profile
is `panic = "abort"`, `Cargo.toml:26`); modular files, one unit per file, with `mod.rs` used only for
re-exports; and the AGPL header at the top of every source file, matching the header on every existing
module. The architecture gate additionally requires HDA to remain a userland capsule that reaches
hardware only through broker MMIO and IRQ in this slice (`README.md:154`).

## Debugging

The first thing to confirm is that the capsule ran. The driver fleet boots the driver under the tag
`DRIVER-HDA` (`src/userspace/init/spawn_plan/drivers_storage.rs:40`), and on a successful spawn the boot
path logs `[DRIVER-HDA] capsule spawned` (`src/userspace/init/capsule_boot/run.rs:29`); the sibling
drivers page records `[DRIVER-HDA]` as that marker (`docs/userland/drivers.md:270`). If the ELF fails to
load, the spawn path emits the debug tag `[DRIVER-HDA] load_elf_executable error:`
(`src/hardware/hda_capsule/spawn.rs:57`).

Setup failures are hard barriers and each maps to a distinct exit code, so a driver that never comes up
tells you where it stopped (`src/error/types.rs:29`):

```
  2  DeviceNotFound             no PCI HDA function matched discovery          discover.rs:32
  3  BrokerCallFailed           a claim/mmio/irq/pci broker call returned <0   setup/*
  4  ControllerResetTimeout     GCTL.CRST never read back set                  controller/reset.rs:35
  5  ImmediateCommandBusy       IRS.BUSY never cleared before a codec verb     controller/immediate.rs:46
  6  ImmediateResponseTimeout   no valid immediate response for a codec verb   controller/immediate.rs:59
  7  UnsupportedController      GCAP or the major version read back as zero    setup/sequence.rs:41
```

Failure modes and where to look:

- No codecs reported. If discovery found the controller and it left reset but `OP_CODEC_MASK` returns a
  zero mask and `OP_CODEC_LIST` a zero count, no `STATESTS` presence bit is set, so no codec was detected
  on the link (`src/controller/codec_probe.rs:36`, `handlers/codec_mask.rs:28`). `OP_CONTROLLER_INFO`
  still answers with the live registers, so read back `STATESTS` there to confirm the controller sees the
  same empty link.
- A codec is present but its ids are zero. A present codec whose `probe_ok` byte is 0 in `OP_CODEC_LIST`
  means the immediate-command `Get Parameter(Vendor ID)` timed out for that address; the presence bit was
  set but the verb round trip failed (`src/controller/codec_probe.rs:47`, `:56`). A codec whose immediate
  interface never clears busy or never returns a valid response at setup fails the whole bring-up with
  exit code 5 or 6 instead (`src/controller/immediate.rs:46`, `:59`).
- The controller never leaves reset. Exit code 4 (`ControllerResetTimeout`) means `GCTL.CRST` was driven
  high but never read back set within the bounded spin, which points at the controller or the BAR0
  mapping, not the wire format (`src/controller/reset.rs:29`).
- A malformed request that is silently ignored. A frame whose magic is not `NHDA` or whose version is not
  1 is dropped inside `decode_request` and answered with `E_INVAL` via the decode-failed path, and any op
  that arrives with a non-zero payload is answered with `E_INVAL` before dispatch, so a client that gets
  no useful reply should check that it is sending a bare 20-byte `NHDA` v1 header
  (`src/protocol/decode.rs:23`, `src/server/runner/run.rs:50`, `src/server/error.rs:29`).

## Source map

```
  src/main.rs                        _start -> setup::run -> server::run
  src/discover.rs                    find_hda: PCI multimedia/HDA subclass match, MMIO BAR0, routed IRQ
  src/setup/sequence.rs              the one-shot bring-up (claim, bus-master, mmio, irq, reset, info, probe)
  src/setup/{claim,pci,mmio,irq}.rs  the individual broker bring-up steps
  src/setup/driver.rs                the Driver struct: handles, regs, codec probe results
  src/controller/reset.rs            leave_reset: drive GCTL.CRST high and wait
  src/controller/info.rs             ControllerInfo::read: global register snapshot + GCAP-derived counts
  src/controller/streams.rs          GCAP bitfield decode (input/output/bidi stream counts, addr64)
  src/controller/stream_layout.rs    layout: computed descriptor offsets 0x80 + i*0x20
  src/controller/immediate.rs        IC/IR/IRS immediate-command verb transport
  src/controller/codec_probe.rs      probe: STATESTS presence + Get Parameter(Vendor ID) per codec
  src/protocol/{header,decode,encode,ops,errno,limits,endpoint}.rs   the NHDA wire format
  src/regs/mmio.rs                   volatile 8/16/32-bit register accessors over BAR0
  src/server/runner/run.rs           the request loop, dispatch, and payload guard
  src/server/runner/poll_irq.rs      mk_irq_poll / mk_irq_ack each loop iteration
  src/server/handlers/               one file per op (health, controller_info, codec_mask, stream_layout, codec_list)
  src/server/error.rs                reply_with_status / reply_decode_failed
  src/handles/                       BrokerHandles: irq, mmio, device grants freed on drop
  src/constants/{regs,pci}.rs        HDA register offsets and PCI class/BAR constants
  Capsule.mk                         slug, handle, ports, capability mask
  src/hardware/hda_capsule/          the kernel-side embed, verified spawn, and client
  src/userspace/init/spawn_plan/drivers_storage.rs   the driver-fleet spawn entry
  docs/subsystems/hardware-broker/   the claim, mmio, irq, and (future) dma grant contracts
```

Every reference above is verified against those trees.

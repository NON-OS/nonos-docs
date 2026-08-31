# The HDA Driver Capsule

`capsule_driver_hda` is the Intel High Definition Audio driver in the NONOS tree: a signed userland
capsule that owns one HDA PCI controller, brings it out of reset, and answers controller, codec, and
stream-layout queries over IPC. It reaches hardware only through the
[hardware broker](../../subsystems/hardware-broker/README.md), never through kernel driver code.

This is no longer the enumeration-only P0 slice an earlier version of this page (and the still-stale
`Cargo.toml:4-7` header comment) describes. The driver discovers the controller, claims it, maps its
register window, binds its interrupt, releases reset, reads the controller-global registers, and probes
each present codec for its vendor and device id, and it also carries a real output path: a CORB/RIRB verb
ring (`src/controller/corb.rs`, `verb.rs`, `immediate.rs`), a codec node graph
(`src/controller/graph.rs`), a buffer descriptor list builder (`src/controller/bdl.rs`), and stream
setup/run (`src/controller/stream_setup.rs`, `stream_run.rs`, `streams.rs`). The service exposes
`OP_PLAY_TONE`, `OP_WRITE_PCM`, `OP_STREAM_START`, and `OP_STREAM_STOP`
(`src/protocol/ops.rs:22`), and the capsule now holds the `Dma` capability its BDL path needs
(mask `0xF8119`, `Capsule.mk:17`). What is honestly NOT established is that audible audio actually comes
out: there is no proof crate for this driver, only two host test binaries (`tests/host/bdl.rs`,
`tests/host/queue.rs`), and neither QEMU nor real-hardware playback is asserted anywhere in the tree. So
the tag is IMPLEMENTED (controller, codec, BDL, stream, PCM/tone), NOT PROVEN (no sample is shown reaching
a speaker). Read every page here with that line in mind, and disregard the `Cargo.toml` header, which
still claims streams are absent.

The source is organized so the documentation can mirror it one page per code pillar, and each page can be
read beside the folder it describes.

## Identity

Everything the kernel and the service registry need to name and reach the driver comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-hda` | `Capsule.mk:7` |
| Service handle | `driver.hda0` | `Capsule.mk:8`, `src/protocol/endpoint.rs:17` |
| Namespace | `systems.nonos.driver.hda0` | `Capsule.mk:13` |
| Service endpoint | `service:4218:driver.hda0` | `Capsule.mk:14`, `src/hardware/hda_capsule/spawn.rs:33` |
| Reply endpoint | `reply:4219:endpoint.4294967312` | `Capsule.mk:15`, `spawn.rs:34` |
| Reply inbox (kernel) | `endpoint.4294967312` (`0x1_0000_0010`) | `src/hardware/hda_capsule/client/transport.rs:25`, `src/protocol/endpoint.rs:18` |
| Capability mask | `0xF8119` | `Capsule.mk:17` |
| Binary name | `driver_hda` | `Capsule.mk:11` |
| Kernel mirror | `src/hardware/hda_capsule` | `src/hardware/hda_capsule/spawn.rs:37` |

The reply endpoint name `endpoint.4294967312` is the decimal spelling of the reply-inbox id the capsule
sends every response to. That id is `KERNEL_REPLY_ENDPOINT = 0x1_0000_0010` in the capsule
(`src/protocol/endpoint.rs:18`) and `REPLY_INBOX = "endpoint.4294967312"` on the kernel side
(`src/hardware/hda_capsule/client/transport.rs:25`); `0x1_0000_0010` is 4294967312, so the two agree.

The mask `0xF8119` decomposes bit by bit against `src/capabilities/types/bit.rs` (the enum lives in
`src/capabilities/types/defs.rs`):

```
  0x00001  CoreExec     bit()      1     types/bit.rs:23
  0x00008  IPC          bit()      8     types/bit.rs:26
  0x00010  Memory       bit()     16     types/bit.rs:27
  0x00100  Debug        bit()    256     types/bit.rs:31
  0x08000  DeviceEnum   bit()  32768     types/bit.rs:38
  0x10000  Driver       bit()  65536     types/bit.rs:39
  0x20000  Mmio         bit() 131072     types/bit.rs:40
  0x40000  Irq          bit() 262144     types/bit.rs:41
  0x80000  Dma          bit() 524288     types/bit.rs:42
  -------
  0xF8119  = 1 + 8 + 16 + 256 + 32768 + 65536 + 131072 + 262144 + 524288
```

Those are the capabilities an MMIO device driver with an interrupt and a DMA-backed stream needs.
`DeviceEnum` is enumerate-only, `Driver` allows claim and release of one device, `Mmio` maps a slice of a
BAR into the capsule's address space, `Irq` binds a device interrupt to a kernel-delivered slot, and `Dma`
allocates the device-visible buffer the BDL and stream descriptors point at. The `Dma` bit is present
precisely because this driver does now program a stream, which is the change from the P0 enumeration-only
slice; the `Debug` bit is the serial-debug grant every driver carries. There is no `Pio` bit (`0x100000`,
`types/bit.rs:43`), no `FileSystem` bit (`64`, `types/bit.rs:29`), no `Network` bit (`4`,
`types/bit.rs:25`), and no `Admin` (`512`, `types/bit.rs:32`). The driver cannot touch an I/O port, read a
file, open a socket, or reach raw kernel memory.

Two identity notes worth recording, because they are places where the code and its neighbours can drift:

- The `Capsule.mk` comment on line 16 lists `IPC|Memory|Driver|DeviceEnum|Mmio|Irq|Dma|Debug = 0xF8119`,
  but that eight-name set sums to `0xF8118`. The extra `0x1` in the real mask is `CoreExec`, which every
  runnable capsule needs and which the comment simply omits. The numeric value `0xF8119` on line 17 is the
  authority signed into the manifest, and it is the one decomposed above.
- The kernel-side spawn record requests only the eight hardware, IPC, DMA, and debug bits and not
  `CoreExec` explicitly (`src/hardware/hda_capsule/spawn.rs:51`). The signed manifest mask `0xF8119`
  (`Capsule.mk:17`) is the ceiling the trust anchor enforces; the spawn `requested_caps` is a request
  bounded by that manifest, and `CoreExec` is granted to the capsule as a matter of being an executable
  process.

## The two pillars

The source under `userland/capsule_driver_hda/src/` splits cleanly into two halves, and the documentation
is one page each. `_start` initialises the heap, runs the one-shot setup sequence once, and hands the
resulting `Driver` to the request loop; if setup fails it exits with a code derived from the error
(`src/main.rs:37`). The two halves never overlap: `setup::run` performs every privileged bring-up step
once and returns a `Driver` that owns the broker grants and the mapped register window, and `server::run`
is an endless request loop that never touches the broker again except to poll and acknowledge the
controller interrupt (`src/setup/sequence.rs:27`, `src/server/runner/run.rs:32`).

```
  setup::run  ->  Driver  ->  server::run
  bring-up        the        the request
  once            grants     loop, forever
```

| Page | Mirrors | What it covers |
|------|---------|----------------|
| [bringup.md](bringup.md) | `src/discover.rs`, `src/setup/`, `src/controller/`, `src/constants/`, `src/regs/` | The one-shot privileged path: discovery, claim, bus-master, BAR0 map, IRQ bind, reset release, controller-info read, and the immediate-command codec probe and `GCAP`-derived stream layout. |
| [operations.md](operations.md) | `src/protocol/`, `src/server/`, `src/handles/` | The wire format, the five query operations and their handlers, the request loop and its guards, the runtime IRQ poll, and the broker-grant owner freed on drop. |
| [contributing.md](contributing.md) | the whole tree | Where to work, how to add an operation, what a playback path would still need, the build and sign targets, and the code standards. |
| [debugging.md](debugging.md) | runtime | The boot marker, the per-error exit codes, and where to look when a codec, the reset, or a request misbehaves. |

## Lifecycle

The capsule is spawned through [verified spawn](../../security/capsules-and-trust.md): its signature and
attestation are checked, its requested capabilities are held against its manifest ceiling, and only then
is its ELF mapped (`src/hardware/hda_capsule/spawn.rs:37`). The driver fleet boots it under the tag
`DRIVER-HDA` (`src/userspace/init/spawn_plan/drivers_storage.rs:40`), and a successful spawn logs
`[DRIVER-HDA] capsule spawned` on the boot path (`src/userspace/init/capsule_boot/run.rs:29`).

Inside the capsule the order is fixed (`src/setup/sequence.rs:27`): discover the HDA function, claim it,
enable bus mastering, map BAR0, bind the interrupt, drive the controller out of reset, read the
controller-global registers, reject an unusable controller, and probe each present codec. The resulting
`Driver` then enters the request loop, which polls the interrupt and answers one query at a time. The
[debugging](debugging.md) page covers what each later marker and exit code means. All broker grants are
owned by one `BrokerHandles` that releases the IRQ bind, the BAR0 mapping, and the device claim in that
order on drop (`src/handles/broker_handles_drop.rs:22`).

## Source map

The pages under this folder are drawn from `userland/capsule_driver_hda/` (the capsule source and its
`Capsule.mk` and `Cargo.toml`), `src/capabilities/types/bit.rs` (the capability bit values), the kernel spawn
mirror under `src/hardware/hda_capsule/`, and the driver-fleet spawn entry under
`src/userspace/init/`. The broker grant contracts this driver relies on are documented under
`docs/subsystems/hardware-broker/`.

```
  userland/capsule_driver_hda/src/main.rs        _start -> setup::run -> server::run
  userland/capsule_driver_hda/src/discover.rs    find_hda: PCI multimedia/HDA match, MMIO BAR0, routed IRQ
  userland/capsule_driver_hda/src/setup/         the one-shot broker bring-up
  userland/capsule_driver_hda/src/controller/    reset, controller-info, codec probe, stream layout
  userland/capsule_driver_hda/src/constants/     HDA register offsets and PCI class/BAR constants
  userland/capsule_driver_hda/src/regs/          volatile 8/16/32-bit accessors over BAR0
  userland/capsule_driver_hda/src/protocol/      the NHDA wire format
  userland/capsule_driver_hda/src/server/        the request loop and per-op handlers
  userland/capsule_driver_hda/src/handles/       BrokerHandles: irq, mmio, device grants freed on drop
  userland/capsule_driver_hda/Capsule.mk         slug, handle, ports, capability mask
  src/capabilities/types/bit.rs                  the capability bit values
  src/hardware/hda_capsule/                      the kernel-side embed, verified spawn, and client
  src/userspace/init/                            the driver-fleet spawn entry and boot marker
  docs/subsystems/hardware-broker/               the claim, mmio, and irq grant contracts
```

Every reference above is verified against those trees.

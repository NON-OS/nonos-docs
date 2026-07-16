# capsule_driver_i2c_hid (full reference)

`capsule_driver_i2c_hid` is the HID-over-I2C class driver in the NONOS tree. On a laptop the device it
speaks to is the I2C peripheral behind the keyboard deck, most often the Precision Touchpad or an I2C
touchscreen. The capsule does not touch the bus hardware itself. It sits one level above the I2C
controller capsule `driver.i2c_pci0`, asks that capsule to run bounded I2C transfers on its behalf, reads
the device's HID descriptor, polls its input register for reports, decodes each report, and posts the
result into the kernel input ring as pointer and button events. This is the exhaustive reference for the
driver as it exists on this branch; be aware that a larger touchpad build exists on another branch and is
called out honestly below.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [The report and input-post path](#the-report-and-input-post-path)
- [Architecture and bring-up](#architecture-and-bring-up)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The capsule is a `no_std`/`no_main` binary. `_start` initialises the heap, runs `setup::run` to resolve
the I2C controller and do a first probe, and then hands the built `State` to the server loop; any setup
failure exits with code 1 (`userland/capsule_driver_i2c_hid/src/main.rs:32`). It holds no hardware
grants. Every byte it exchanges with the physical device goes out as an IPC request to `driver.i2c_pci0`,
which owns the controller registers and the interrupt line and is the only capsule in the pair with MMIO
and IRQ authority (`README.md:9`, `src/i2c_client/service.rs:4`).

Two things run in one loop. The capsule is a small IPC server, answering health, reprobe, and descriptor
queries from other capsules, and on every pass through that loop it also polls the device's input
register, decodes whatever report came back, and posts input events. The report path is where the real
work happens: it reads the HID input register over I2C, parses the returned bytes into a pointer sample,
and turns motion, wheel, and button changes into kernel input events
(`src/server/runner.rs:27`, `src/input/poll.rs:22`).

The honest scope of this branch: the report parser decodes a relative-pointer report (buttons, dx, dy,
wheel), not the absolute multi-contact Precision Touchpad report format
(`src/input/parse_report.rs:19`). Pacing is a short receive timeout on the server loop, not a hardware
interrupt or GPIO doorbell (`src/server/runner.rs:13`). A separate branch carries the absolute-mode
touchpad decode and the doorbell-paced read; that work is not present in the source documented here. See
[Architecture and bring-up](#architecture-and-bring-up) for the specifics.

## Identity

Everything the kernel and the service registry need to name and reach the driver comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-i2c-hid` | `Capsule.mk:5` |
| Service handle | `driver.i2c_hid0` | `Capsule.mk:6`, `src/userspace/capsule_driver_i2c_hid/spawn.rs:23` |
| Namespace | `systems.nonos.driver.i2c_hid0` | `Capsule.mk:11` |
| Service endpoint | `service:4232:driver.i2c_hid0` | `Capsule.mk:12`, `spawn.rs:24` |
| Reply endpoint | `reply:4233:endpoint.4294967319` | `Capsule.mk:13`, `spawn.rs:25` |
| Capability mask | `0x200019` | `Capsule.mk:14` |
| Binary name | `driver_i2c_hid` | `Capsule.mk:9` |
| Kernel mirror | `src/userspace/capsule_driver_i2c_hid` | `Capsule.mk:15` |

The mask `0x200019` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x000001  CoreExec       bit()       1   types.rs:55
  0x000008  IPC            bit()       8   types.rs:58
  0x000010  Memory         bit()      16   types.rs:59
  0x200000  InputSource    bit() 2097152   types.rs:77
  --------
  0x200019  = 1 + 8 + 16 + 2097152
```

The kernel spawn path requests exactly IPC, Memory, and InputSource and no others; CoreExec is the
implicit execute bit every capsule carries (`spawn.rs:42`). `InputSource` is the one capability that
separates this driver from an ordinary IPC capsule: it is the bit the kernel checks before it will accept
an input-event post (`src/syscall/contract/cap_table/mk.rs:78`), so it is the sole authority behind the
report path. There is no `Driver`, `Mmio`, `Irq`, `Dma`, `Pio`, `DeviceEnum`, `Network`, `FileSystem`, or
graphics bit in the mask. The driver cannot claim a device, map a BAR, bind an interrupt, or run a port
instruction. It can speak IPC, hold memory, and post input, and nothing else.

## Operation reference

The server exposes exactly three operations. Each is dispatched only when its request body is empty; a
recognised op with a non-empty body is rejected, and an unrecognised op is rejected separately
(`src/server/runner.rs:43`). Ops and error codes come from `src/protocol/ops.rs` and
`src/protocol/errno.rs`.

| Op | Value | Request | Reply | Handler |
|---|---|---|---|---|
| `OP_HEALTHCHECK` | 1 | empty | found flag, address, ports, pid, and the probe/poll/report/failure counters | `ops.rs:1`, `server/handlers/health.rs:5` |
| `OP_PROBE` | 2 | empty | re-runs the bus probe, replies with the found flag and selected address | `ops.rs:2`, `server/handlers/probe.rs:6` |
| `OP_DESCRIPTOR` | 3 | empty | selected address and the cached HID descriptor bytes, or `E_NOT_FOUND` | `ops.rs:3`, `server/handlers/descriptor.rs:5` |

Reply detail:

- `OP_HEALTHCHECK` always succeeds with `E_OK` and a 56-byte body: the found flag, the I2C address, the
  controller port and pid, the probe count, the discovered input register and input length, and the
  running `input_polls`, `input_reports`, and `post_failures` counters
  (`src/server/handlers/health.rs:6`, `src/state.rs:9`). This is the single introspection surface for the
  report path, since those three counters are the only externally visible sign that polling is happening.
- `OP_PROBE` calls `reprobe`, which bumps the probe counter and re-runs the address scan, then replies
  with the found flag and the selected address (`src/server/handlers/probe.rs:7`, `src/setup.rs:12`).
- `OP_DESCRIPTOR` returns `E_NOT_FOUND` when no descriptor has been read yet (`found()` is false), and
  otherwise replies `E_OK` with the address and the descriptor bytes trimmed to `descriptor_len`
  (`src/server/handlers/descriptor.rs:6`).

Error codes: `E_OK 0`, `E_NOT_FOUND -2`, `E_INVAL -22`, `E_BAD_OP -38`
(`src/protocol/errno.rs:1`). A recognised op carrying a non-empty body draws `E_INVAL`; an op the server
does not know draws `E_BAD_OP` (`src/server/runner.rs:49`).

## The report and input-post path

The report path does not run in response to an operation. It runs on every iteration of the server loop,
right after the bounded receive returns, whether or not a request arrived (`src/server/runner.rs:27`).
`input::poll` is the whole path (`src/input/poll.rs:22`):

1. It returns immediately unless a descriptor was found and a usable input register and input length were
   parsed from it (`state.found()`, `input_register != 0`, `input_len >= 5`)
   (`src/input/poll.rs:23`).
2. It builds the two-byte little-endian input-register address, bumps `input_polls`, and asks the I2C
   controller to write that register address and read back up to `input_len` bytes (capped at the 64-byte
   local buffer) through `write_read` (`src/input/poll.rs:26`).
3. It parses the returned bytes with `parse_report`. The report is length-prefixed: the first two bytes
   are the total report length, and the body is what follows. If the body looks like it carries a leading
   report id, the parser skips that one id byte, then reads buttons (low five bits), dx, dy, and an
   optional wheel byte as signed eight-bit values into a `MouseSample`
   (`src/input/parse_report.rs:19`, `src/input/sample.rs:17`).
4. On a good parse it bumps `input_reports` and calls `publish` (`src/input/poll.rs:31`).

`publish` is where events reach the kernel (`src/input/publish.rs:25`). It posts a relative-pointer event
when dx or dy is non-zero, a wheel event when the wheel byte is non-zero, and, by comparing the new
button bits against `last_buttons`, one button-down or button-up event per changed bit across the low
five buttons (codes 1..5). Each of those is a call to `post`, which fills an `InputEvent` and hands it to
the `mk_input_event_post` syscall (`src/input/post.rs:19`). If any post is refused the capsule bumps
`post_failures` (`src/input/publish.rs:43`); it never retries and never blocks the loop on a failed post.
The event kinds are the shared libc constants `INPUT_KIND_POINTER_REL`, `INPUT_KIND_WHEEL`,
`INPUT_KIND_BUTTON_DOWN`, and `INPUT_KIND_BUTTON_UP`
(`userland/libc/src/surface_registry/types.rs:25`).

From there the event is out of the capsule's hands. `mk_input_event_post` lands on the kernel's
`post_input`, which pushes the event into the bounded MPSC input ring that the input router capsule
drains (`src/kernel_core/surface_registry/input_ring.rs:55`). The full event journey, from a driver post
through the ring to the router and on to the focused surface, is documented in
[the input path](../../subsystems/input/path.md).

## Architecture and bring-up

The capsule is four working modules plus the shared state: `hid` (descriptor discovery), `i2c_client`
(the client side of the I2C controller protocol), `input` (the report and post path), `protocol` (this
capsule's own IPC wire format), and `server` (the request loop) (`src/main.rs:23`).

Bring-up runs in `setup::run` (`src/setup.rs:5`):

1. Resolve the controller. It looks up the service name `driver.i2c_pci0` in the registry through
   `mk_service_lookup`; a missing controller means startup fails closed and the process exits
   (`src/i2c_client/service.rs:4`, `src/main.rs:38`). This is how the driver "finds" the bus: not by ACPI
   enumeration in this capsule, but by resolving the controller capsule that already owns the claimed I2C
   host. ACPI discovery of the I2C host and its HID device lives in the controller and the kernel broker,
   not here.
2. Probe for the device. `reprobe` scans a fixed list of common HID-over-I2C slave addresses
   (`0x10, 0x15, 0x2C, 0x38, 0x4B, 0x4C, 0x20, 0x24`, covering ELAN, Synaptics, and FocalTech ranges),
   and for each one writes the HID descriptor register `0x0001` and reads back 30 bytes
   (`src/hid/probe.rs:4`, `src/hid/probe.rs:9`).
3. Validate the descriptor. A candidate is accepted only if the 30-byte read parses as a HID descriptor:
   the descriptor-length field must be in `28..=256` and the BCD version field in `0x0100..=0x0111`
   (`src/hid/descriptor.rs:3`). The first address that validates wins; its address and the descriptor are
   stored.
4. Derive the report registers. From the accepted descriptor the capsule reads the input register at
   descriptor offset 8..10 and the maximum input length at offset 10..12 (capped at 64)
   (`src/hid/input_register.rs:17`, `src/hid/input_len.rs:17`). Those two values are what arm the report
   path in step 1 of `poll`.

The report protocol on the wire is HID-over-I2C's register model: write a little-endian register address,
then read the report bytes back, both carried inside the controller's `OP_TRANSFER` envelope
(`src/i2c_client/transfer.rs:7`). A write-then-read transfer sets a restart-on-read flag so the controller
issues a repeated start rather than a stop between the phases (`src/i2c_client/wire.rs:22`).

Pacing, honestly. The read cadence is set by the server loop's receive timeout: `mk_ipc_recv_from` is
called with a 2 ms timeout, so if no request arrives the loop wakes roughly every 2 ms and polls the
device (`src/server/runner.rs:13`, `src/server/runner.rs:20`). There is no interrupt binding and no GPIO
doorbell in this build; the capsule holds no `Irq` capability, so it cannot bind the device interrupt.
This is a real gap against a production touchpad driver, which would read only when the device signals a
report ready.

Known partial-support gaps, stated plainly:

- The report decode is a relative-pointer format, not the absolute multi-contact Precision Touchpad
  report. A real Precision Touchpad in its native mode will not decode correctly here; the parser reads
  the first contact's fields as if they were mouse deltas (`src/input/parse_report.rs:19`).
- No input-mode switch. The driver never writes the feature report that puts a Precision Touchpad into
  absolute mode; it reads whatever mode the device powers up in.
- No interrupt or doorbell pacing, as above.
- No gesture recognition, palm rejection, keyboard-layout mapping, or power management. The README lists
  these as explicit non-goals for the current slice (`README.md:117`).

A separate branch carries the fuller touchpad path (absolute-mode switch, a report-descriptor parser that
locks onto the touch report id, GPIO-doorbell-paced reads, and gesture handling). That code is not on
this branch and none of the file:line references above point at it; treat this page as the reference for
the driver as it is shipped here.

## Protocol and IPC

The driver speaks two protocols: its own server protocol, which other capsules call, and the I2C
controller's client protocol, which it calls.

Server protocol, magic `NHID` `0x4E484944`, version 1, 20-byte header
(`src/protocol/header.rs:1`). A request is `magic(4) | version(2) | op(2) | request_id(8) | body_len(4)`
followed by the body; `parse` rejects anything with the wrong magic, wrong version, or a truncated body
(`src/protocol/decode.rs:3`). A reply is the same header with the request's op and id, a four-byte signed
status word, and the body (`src/protocol/encode.rs:3`). The loop receives with `mk_ipc_recv_from`,
ignores empty or unsourced deliveries, and sends every reply through `mk_ipc_reply`
(`src/server/runner.rs:20`, `src/server/respond.rs:6`). The payload cap is 96 bytes
(`src/protocol/limits.rs:1`).

I2C controller client protocol, magic `NI2C` `0x4E493243`, version 1, 20-byte header, `OP_TRANSFER 5`
(`src/i2c_client/wire.rs:1`). `write_read` allocates a request buffer, encodes the slave address, the
write bytes, the requested read length, and the restart-on-read flag, then makes a blocking call to the
controller's port with a 250 ms timeout through `mk_ipc_call_timeout`
(`src/i2c_client/transfer.rs:7`). The reply is checked for the right magic, a matching request id, and a
zero status before its payload is copied out; anything else returns `None` and the transfer is treated as
having failed (`src/i2c_client/wire.rs:28`). Request ids are handed out by a monotonic atomic counter so a
stale reply for an earlier transfer is rejected (`src/i2c_client/seq.rs:5`).

So the answer to "does it talk to an i2c-pci controller capsule": yes, exclusively. Every descriptor read
and every report read is an `OP_TRANSFER` call to `driver.i2c_pci0`; the driver never touches a register
directly (`src/hid/probe.rs:2`, `src/input/poll.rs:17`).

The input-post syscall it calls is `mk_input_event_post`, which the kernel gates on the `InputSource`
capability (`src/input/post.rs:17`, `src/syscall/contract/cap_table/mk.rs:78`). That is the only syscall
the capsule uses that carries hardware-adjacent authority, and it is the reason `InputSource` is in the
mask.

## Security analysis

This driver posts input, which is a sensitive authority: an event in the ring can move the pointer, click
a button, and, in the general input path, deliver keystrokes to whatever surface has focus. The design
answer is to grant the driver that one authority and withhold everything else. Its mask `0x200019` is
CoreExec, IPC, Memory, and InputSource, and no hardware capability at all (`Capsule.mk:14`, `spawn.rs:42`).
It cannot claim a device, map MMIO, bind an IRQ, run DMA, or issue a port instruction, so a bug in the
report parser or the I2C client cannot reach the bus hardware or any other device; the worst it can do on
the hardware side is send a malformed `OP_TRANSFER` to the controller, which validates and bounds every
transfer itself.

The one authority it does hold, `InputSource`, is checked in the kernel on every post: `MkInputEventPost`
is refused unless the caller's capability set includes it (`src/syscall/contract/cap_table/mk.rs:78`), and
the post lands in a bounded ring that drops rather than overflows (`input_ring.rs:59`). The driver posts
only what it decodes: relative pointer motion, a wheel delta, and button transitions across five buttons,
with no absolute coordinates and no key events in this build (`src/input/publish.rs:25`). A compromised or
buggy driver could inject spurious pointer and button events, which is inherent to any input driver
holding `InputSource`; the containment is that this is all it can do, and the router and the focused
surface still apply their own policy downstream.

Isolation and privacy. The capsule keeps only volatile state: the controller port and pid, the selected
I2C address, the 30-byte descriptor, the derived input register and length, the last button bitmap, and a
handful of counters (`src/state.rs:1`). It stores no report history, no gesture state, and no keystrokes,
and it has no filesystem or network capability to persist or exfiltrate anything even if it wanted to
(`README.md:46`). Cross-capsule isolation is the kernel's: the driver is a CPL 3 user binary, verified and
enrolled at spawn like every other capsule, and it reaches the controller only by resolving a service
name, not by holding a handle to the controller's memory.

## How to contribute

The source lives at `userland/capsule_driver_i2c_hid/`. The descriptor discovery is under `src/hid/`, the
I2C controller client under `src/i2c_client/`, the report and post path under `src/input/`, the server
wire format under `src/protocol/`, and the request loop under `src/server/`. The kernel-side embed and
verified spawn are mirrored at `src/userspace/capsule_driver_i2c_hid/`.

To extend the report decode, the module to edit is `src/input/parse_report.rs`, which owns the mapping
from raw report bytes to a `MouseSample`, and `src/input/publish.rs`, which owns the mapping from a
`MouseSample` to posted events. Adding a new report layout means teaching `parse_report` to recognise it
and, if it carries fields the current `MouseSample` does not hold, extending that struct in
`src/input/sample.rs`. To add a server operation, add the constant in `src/protocol/ops.rs`, add a handler
under `src/server/handlers/`, wire it into the match in `src/server/runner.rs:43`, and re-export it from
`src/server/handlers/mod.rs`.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_driver_i2c_hid/Capsule.mk:17`):

```
  make nonos-mk-driver-i2c-hid               build the capsule ELF
  make nonos-mk-driver-i2c-hid-sign          produce the id cert, manifest, and attestation trailer
  make nonos-mk-driver-i2c-hid-verify        verify the signed artifacts against the trust anchor
  make nonos-mk-check-driver-i2c-hid-keys    check the per-capsule signing keys exist
```

For a kernel image that includes this driver, `make nonos-mk-driver-i2c-hid-prod` builds the
`microkernel-driver-i2c-hid` profile, which pulls in the proof-io, i2c-pci, and i2c-hid artifacts
together (`Makefile:965`). The README also documents a one-shot rebuild `make -B nonos-mk-driver-i2c-hid`
and the profile check `cargo check --no-default-features --features microkernel-driver-i2c-hid`
(`README.md:125`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every failure returns through an `Option`/`Result` or a pushed error status,
and the release profile is `panic = "abort"`, `Cargo.toml:18`); modular files, one unit per file, with
`mod.rs` used only for re-exports (as `src/hid/mod.rs` and `src/protocol/mod.rs` already are); and the
AGPL header at the top of every source file. Note that a few of the older files under `src/hid/`,
`src/protocol/`, `src/server/`, and `src/i2c_client/` predate the header sweep and do not yet carry it
(for example `src/hid/descriptor.rs:1` and `src/protocol/header.rs:1` open with code, while the newer
`src/input/*` files carry the full header); any new file must include the header, and touching an
old one is a good chance to add it.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[DRIVER-I2C-HID] capsule spawned` (tag `DRIVER-I2C-HID`, message `capsule spawned`) from the boot log
(`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`); the spawn is driven from
the bus-driver plan (`src/userspace/init/spawn_plan/drivers_bus.rs:52`). An absent line means the capsule
never started, usually a signature, manifest, or capability failure, and the boot log prints an `[ERROR]`
line instead (`src/userspace/init/capsule_boot/run.rs:32`).

The live introspection surface is `OP_HEALTHCHECK`, whose reply carries the found flag, the selected
address, and the `probes`, `input_polls`, `input_reports`, and `post_failures` counters
(`src/server/handlers/health.rs:6`). Those counters isolate most failures.

Failure modes and where to look:

- Startup fails closed, no capsule at all. `setup::run` returns an error only when `driver.i2c_pci0`
  does not resolve (`src/setup.rs:6`). Confirm the I2C controller capsule is spawned and registered
  before this one; without it, `_start` exits 1 (`src/main.rs:38`).
- Touchpad dead, descriptor never found. `found()` stays false when no candidate address returns a valid
  30-byte descriptor (`src/hid/probe.rs:6`). `OP_DESCRIPTOR` returns `E_NOT_FOUND` and `OP_HEALTHCHECK`
  shows the found flag clear with a rising `probes` count. The device address may be outside the fixed
  candidate list (`src/hid/probe.rs:4`), or the descriptor may fail the length or BCD-version check
  (`src/hid/descriptor.rs:3`). `OP_PROBE` forces a re-scan.
- Descriptor found but no reports. `OP_HEALTHCHECK` shows `found` set but `input_reports` flat. If
  `input_polls` is also flat, the poll guard is rejecting the setup: the derived input register is zero or
  the input length is under five bytes (`src/input/poll.rs:23`). If `input_polls` climbs while
  `input_reports` stays flat, the transfer is failing or the returned bytes are not parsing: `write_read`
  returned `None` (a controller error or timeout) or `parse_report` rejected the length prefix
  (`src/input/poll.rs:30`, `src/input/parse_report.rs:20`).
- Reports arrive but the pointer misbehaves or coordinates are wrong. This is the expected symptom on a
  real Precision Touchpad, because the parser decodes a relative-pointer layout, not the absolute
  multi-contact report (`src/input/parse_report.rs:19`). The fix is a proper report decode, not a tweak;
  see the partial-support gaps above.
- Events decode but never reach the surface. If `post_failures` climbs, the kernel is refusing the post,
  which points at the `InputSource` capability check (`src/syscall/contract/cap_table/mk.rs:78`) or a full
  input ring (`src/kernel_core/surface_registry/input_ring.rs:59`). If `post_failures` stays flat and
  events still do not appear, the input path past the ring is the suspect, not this driver; see
  [the input path](../../subsystems/input/path.md).

## Source map

```
  src/main.rs                         _start -> setup::run -> server::run
  src/setup.rs                        resolve controller, first probe, reprobe
  src/state.rs                        State: port, pid, address, descriptor, input reg/len, counters
  src/hid/probe.rs                    candidate-address scan, descriptor register read
  src/hid/descriptor.rs               30-byte descriptor validation (length, BCD version)
  src/hid/input_register.rs           input register from descriptor offset 8..10
  src/hid/input_len.rs                input length from descriptor offset 10..12 (capped 64)
  src/i2c_client/service.rs           resolve driver.i2c_pci0 via mk_service_lookup
  src/i2c_client/transfer.rs          write_read: OP_TRANSFER call with 250 ms timeout
  src/i2c_client/wire.rs              NI2C wire encode/decode, restart-on-read flag
  src/i2c_client/seq.rs               monotonic request-id counter
  src/input/poll.rs                   the report path: read input register, parse, publish
  src/input/parse_report.rs           length-prefixed report -> MouseSample (relative layout)
  src/input/publish.rs                MouseSample -> pointer/wheel/button events, post_failures
  src/input/post.rs                   mk_input_event_post wrapper
  src/protocol/                       NHID server wire format, ops, errno, limits
  src/server/runner.rs                the recv/poll/dispatch loop (2 ms recv timeout)
  src/server/handlers/                health, probe, descriptor op handlers
  Capsule.mk                          slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_driver_i2c_hid/spawn.rs   the kernel-side verified spawn and cap request
  src/userspace/init/spawn_plan/drivers_bus.rs    the bus-driver spawn entry
  userland/libc/src/surface_registry/             mk_input_event_post and the INPUT_KIND_* constants
  nonos-mk/capsule.mk                 the generated nonos-mk-driver-i2c-hid[-sign|-verify] targets
```

Every reference above is verified against those trees.

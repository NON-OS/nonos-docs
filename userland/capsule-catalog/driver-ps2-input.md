# capsule_driver_ps2_input (full reference)

`capsule_driver_ps2_input` is the userland owner of the legacy i8042 PS/2 controller. It drives the
keyboard on IRQ1 and the AUX mouse on IRQ12 from one CPL 3 capsule, because both devices share the
data and command ports at `0x60` and `0x64` and a single controller must have a single owner. It reads
and writes those ports only through the hardware broker's port-IO grant, decodes scancodes and mouse
packets into kernel input events, and exposes bounded event rings and diagnostics over its IPC service.
This is the exhaustive reference; the input path overview is in
[../../subsystems/input/path.md](../../subsystems/input/path.md).

It is a broker-driven device capsule, not a GUI app. The kernel mirror spawns it under service handle
`driver.ps2_kbd0` on service port 4208 with a reply port on 4209, and its capability mask is `0x358019`
(`userland/capsule_driver_ps2_input/Capsule.mk:17`). The source is
`userland/capsule_driver_ps2_input/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [Scancode and mouse decode](#scancode-and-mouse-decode)
- [Architecture and bring-up](#architecture-and-bring-up)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The capsule is `no_std`/`no_main`. `_start` initialises the heap, then loops calling `setup::run`
until the broker bring-up succeeds, yielding between attempts, and finally hands the resulting `Driver`
to the server loop (`src/main.rs:31`). `setup::run` discovers and claims the i8042 platform records,
mints the PIO grant, binds IRQ1 and IRQ12, brings the controller up, and returns the grant ids and a
mouse-enabled flag (`src/setup/sequence.rs:26`).

Once running, the server does two things in one loop. It drains the controller's output buffer into two
bounded rings, a keyboard ring and a mouse ring, translating each byte as it arrives; and it answers
IPC requests on the driver endpoint by replying with liveness, drained events, diagnostic counters, or a
controller snapshot (`src/server/runner.rs:33`). As it decodes a key or a mouse packet it also posts a
kernel input event through `mk_input_event_post`, which is what the compositor and the input router
consume (`src/keymap/post.rs:53`, `src/mouse/post.rs:52`). The IPC event rings and the kernel input
ring are two separate delivery paths from the same decoded stream: the rings are pollable diagnostics,
and the input post is the live path into the window system.

The capsule owns the i8042 PIO grant, the IRQ1 and IRQ12 grants, the keyboard decoder state, the mouse
packet parser state, and both bounded rings. It does not own focus, routing, layout, keyboard layout
policy, cursor policy, or any persistence; those belong to higher userland capsules
(`userland/capsule_driver_ps2_input/README.md:227`).

## Identity

Everything the kernel and the service registry need to name and reach the driver comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-ps2-input` | `Capsule.mk:6` |
| Service handle | `driver.ps2_kbd0` | `Capsule.mk:7`, `src/hardware/ps2_kbd_capsule/spawn.rs:32` |
| Namespace | `systems.nonos.driver.ps2_kbd0` | `Capsule.mk:12` |
| Service endpoint | `service:4208:driver.ps2_kbd0` | `Capsule.mk:13`, `spawn.rs:33` |
| Reply endpoint | `reply:4209:endpoint.4294967306` | `Capsule.mk:14`, `spawn.rs:34` |
| Capability mask | `0x358019` | `Capsule.mk:17` |
| Binary name | `driver_ps2_input` | `Capsule.mk:10` |
| Kernel mirror | `src/hardware/ps2_kbd_capsule` | `Capsule.mk:18` |

The service name is kept as `driver.ps2_kbd0` for compatibility with older callers even though the
endpoint now serves both keyboard and mouse (`README.md:11`). The mask `0x358019` decomposes bit by bit
against `src/capabilities/types.rs`:

```
  0x000001  CoreExec       bit()       1    types.rs:56
  0x000008  IPC            bit()       8    types.rs:59
  0x000010  Memory         bit()      16    types.rs:60
  0x008000  DeviceEnum     bit()   32768    types.rs:71
  0x010000  Driver         bit()   65536    types.rs:72
  0x040000  Irq            bit()  262144    types.rs:74
  0x100000  Pio            bit()  1048576   types.rs:76
  0x200000  InputSource    bit()  2097152   types.rs:77
  --------
  0x358019  = 1 + 8 + 16 + 32768 + 65536 + 262144 + 1048576 + 2097152
```

The kernel spawn path requests exactly those eight capabilities and no others
(`src/hardware/ps2_kbd_capsule/spawn.rs:51`). `InputSource` is what lets the capsule post decoded key
and pointer events into the kernel input ring; `Pio` mints the i8042 port-IO grant; `Irq` binds and
acknowledges IRQ1 and IRQ12; `DeviceEnum` lists the platform records and `Driver` claims them. There is
no `Mmio` bit (0x20000), no `Dma` bit (0x80000), no `FileSystem`, `Network`, `Admin`, or `Debug`
authority in the mask (`README.md:67`), which is the basis of the security analysis below.

## Operation reference

Requests carry the `NKBD` capsule header (`MAGIC = 0x4E4B4244`, `VERSION = 1`, `HDR_LEN = 20`) and no
payload; the decoder rejects a wrong magic, a wrong version, or a short buffer, and the runner rejects
any request whose `payload_len` is non-zero (`src/protocol/header.rs:16`, `src/protocol/decode.rs:17`,
`src/server/runner.rs:64`). Every reply begins with the 20-byte response header followed by a
little-endian signed 32-bit status word; status `0` means the operation completed
(`src/protocol/encode.rs:17`, `src/protocol/errno.rs:16`). A malformed request is answered `E_INVAL`
(-22) and an unknown op is answered `E_INVAL` as well (`src/server/runner.rs:59`, `:74`).

The five ops are dispatched in `run` (`src/server/runner.rs:68`):

| Op | Value | What it does | Handler |
|---|---|---|---|
| `OP_HEALTHCHECK` | `0x0001` | reply with status only, for liveness | `ops.rs:16`, `handlers/health.rs:18` |
| `OP_POLL_EVENTS` | `0x0002` | drain the keyboard ring, reply `u32 count` then `count` 3-byte records | `ops.rs:17`, `handlers/poll.rs:23` |
| `OP_GET_STATE` | `0x0003` | reply seven little-endian `u64` diagnostic counters | `ops.rs:18`, `handlers/state.rs:22` |
| `OP_CONTROLLER_STATUS` | `0x0004` | reply a 28-byte i8042 status snapshot without consuming a data byte | `ops.rs:19`, `handlers/controller_status.rs:27` |
| `OP_POLL_MOUSE` | `0x0005` | drain the mouse ring, reply `u32 count` then `count` 8-byte records | `ops.rs:20`, `handlers/mouse.rs:23` |

`OP_POLL_EVENTS` first drains the ports, acknowledges IRQ1 and IRQ12, then pops up to `MAX_POLL_EVENTS`
(256) events; each keyboard record is scancode byte, event flags, and a reserved zero
(`src/server/handlers/poll.rs:42`, `src/protocol/limits.rs:17`). `OP_POLL_MOUSE` does the same and packs
each mouse record as `i16 dx`, `i16 dy`, `i8 wheel`, `u8 buttons`, `u8 flags`, reserved zero
(`src/server/handlers/mouse.rs:38`). `OP_GET_STATE` reports, in order, keyboard events seen, keyboard
events dropped, controller parity errors, controller timeout errors, mouse events seen, mouse events
dropped, and mouse packet sync errors (`src/server/handlers/state.rs:27`). `OP_CONTROLLER_STATUS` reads
only the status port, so it cannot swallow a pending key or mouse byte, and reports the raw status byte,
the output-full, parity, and timeout bits, the keyboard ring depth, head, and tail, whether the current
output byte is AUX, whether the mouse was enabled at setup, and the mouse ring depth
(`src/server/handlers/controller_status.rs:28`).

Replies are sent to the kernel reply endpoint `0x1_0000_000A`, not back through the recv socket
(`src/protocol/endpoint.rs:16`, used in every handler, for example `handlers/poll.rs:51`).

## Scancode and mouse decode

Keyboard decode happens as bytes are drained. `absorb` handles the Scan Code Set 1 stream: `0xE0` and
`0xE1` are latched as pending prefix flags, and the next real byte is pushed onto the keyboard ring as a
raw `Event { scancode, flags }` where the flags carry `FLAG_BREAK` (high bit `0x80` set), `FLAG_E0_PREFIX`,
or `FLAG_E1_PREFIX` (`src/poll/absorb.rs:22`, `src/ring/flags.rs:16`). The 3-byte record returned over
`OP_POLL_EVENTS` is this raw scancode plus flags; downstream layout policy is not this capsule's job.

Alongside the raw ring push, `absorb` translates the byte to a keycode and posts it to the kernel input
ring. `translate` strips the break bit, masks the scancode to 7 bits, and looks the key up either in the
E0 table or the base Set 1 table (`src/keymap/translate.rs:23`). The base table is split by range:
`0x01..=0x1D` maps the top rows and left control, `0x1E..=0x39` the home row through space and the left
shift and alt, and `0x3A..=0x58` caps lock, the function keys, and the numpad
(`src/keymap/set1/base.rs:18`, `set1/left.rs:18`, `set1/right.rs:18`, `set1/function.rs:22`). Printable
keys map to their ASCII code; named keys map to keycode constants such as `KEYCODE_LSHIFT = 0x1003` and
`KEYCODE_F1 = 0x1101` (`src/keymap/set1/keycodes.rs:16`). The E0 table maps the extended block: right
control and alt, the arrow cluster, Home/End/PageUp/PageDown, Insert, Delete, and the meta keys
(`src/keymap/set1_e0.rs:21`).

`absorb` maintains a modifier bitmask. When a translated key is a modifier, its bit is set on press and
cleared on release; `modifier_bit` maps the shift, control, alt, and meta keycodes to `MOD_SHIFT=1`,
`MOD_CTRL=2`, `MOD_ALT=4`, `MOD_META=8` (`src/poll/absorb.rs:44`, `src/keymap/post.rs:31`). Every
translated key is then posted through `publish`, which builds an `InputEvent` with kind
`INPUT_KIND_KEY_DOWN` or `INPUT_KIND_KEY_UP`, the keycode in `code`, and the current modifier mask in
`flags`, and hands it to `mk_input_event_post` (`src/keymap/post.rs:41`). Note there is no shared
external keymap crate: the layout tables are internal to this capsule under `src/keymap/set1/`, and the
`MOD_*` values are chosen to match the app-side contract and the USB HID driver's encoding by comment
(`src/keymap/post.rs:24`).

Mouse decode is a standard 3-byte PS/2 packet assembler. `MouseParser::absorb` refuses a first byte that
does not have bit 3 set (the sync bit), counting a sync error rather than desynchronising, then collects
three bytes and parses them (`src/mouse/parser.rs:29`). `parse` extracts the left, right, and middle
button bits, the X and Y overflow flags, and sign-extends the two movement bytes using the sign bits in
byte 0; Y is negated so screen-positive is upward (`src/mouse/packet.rs:23`). The parsed `MouseEvent`
goes on the mouse ring; the wheel delta `dz` is always zero for the base 3-byte protocol
(`src/mouse/event.rs:16`). The pump loop drains the mouse ring and publishes each event to the kernel
input ring: relative motion as `INPUT_KIND_POINTER_REL`, a non-zero wheel as `INPUT_KIND_WHEEL`, and
each changed button as `INPUT_KIND_BUTTON_DOWN` or `INPUT_KIND_BUTTON_UP` with `code = bit + 1` so left
is 1, right is 2, middle is 3 (`src/mouse/post.rs:24`, `:43`). Posting happens once, from the pump loop,
so an event is not delivered twice (`src/mouse/parser.rs:26`).

## Architecture and bring-up

`setup::run` performs an ordered bring-up that never leaves partial ownership behind
(`src/setup/sequence.rs:26`):

1. Discover the PS/2 keyboard platform record. `find_ps2_kbd` lists ACPI-bus records and matches the
   PnP vendor/device `0x0001`/`0x0303`, requiring a BAR because the keyboard record owns the shared PIO
   window; failure returns `ps2 keyboard not present in device list`
   (`src/discover.rs:26`, `src/constants/pnp.rs:16`).
2. Claim the keyboard record with `mk_device_claim`, capturing the claim epoch (`src/setup/claim.rs:17`).
3. Grant the i8042 PIO window over BAR index 0 with `mk_pio_grant`; on failure the device claim is
   released (`src/setup/pio.rs:18`).
4. Bind IRQ1 with `mk_irq_bind` against the keyboard record's `irq_line`; on failure the PIO grant is
   released (`src/setup/irq.rs:19`, `:29`).
5. Discover, claim, and bind the AUX record on IRQ12 through `setup_aux`. The AUX record uses vendor/device
   `0x0001`/`0x0304` and needs no BAR; if anything in the AUX path fails it releases the AUX device claim
   and returns grant id 0, and the keyboard path continues without the mouse
   (`src/setup/setup_aux.rs:21`, `src/discover.rs:29`).
6. Flush stale controller output before enabling anything: read the status port and drain up to 16 bytes
   from the data port while the output-full bit is set (`src/init/flush_output.rs:20`).
7. Enable the keyboard. This is where the controller config byte is handled carefully.
8. Enable the mouse if the AUX IRQ bound. `mouse_enabled` is true only when the AUX grant is non-zero and
   the mouse enable sequence acknowledges (`src/setup/sequence.rs:40`).
9. Acknowledge both IRQ lines once through `open_line` so the first real edge is delivered
   (`src/setup/open_line.rs:17`), then emit the ready marker.

The keyboard enable sequence sends `CTL_ENABLE_KBD` (`0xAE`), flushes the output buffer, reads the config
byte with `CTL_READ_CONFIG` (`0x20`), sets `CONFIG_IRQ1` and clears `CONFIG_KBD_DISABLE`, writes it back
with `CTL_WRITE_CONFIG` (`0x60`), issues a keyboard reset, and enables scanning with `0xF4`
(`src/init/enable_keyboard/enable.rs:26`, `src/constants/ports.rs:18`). The flush after enable is a real
hardware fix: enabling the keyboard can leave an ACK or a stray scancode in port `0x60`, and without the
drain the read after `CTL_READ_CONFIG` would return that leftover byte, so writing it back could clear
`CONFIG_IRQ1` or set the disable bit and kill the keyboard (`src/init/enable_keyboard/enable.rs:28`).

The mouse enable sequence sends `CTL_ENABLE_AUX` (`0xA8`), flushes, reads the config byte, sets both
`CONFIG_IRQ1` and `CONFIG_IRQ12` and clears `CONFIG_AUX_DISABLE`, writes it back, then sends the mouse
`MOUSE_SET_DEFAULTS` (`0xF6`) and `MOUSE_ENABLE_REPORTING` (`0xF4`) commands, each routed to the AUX port
through `CTL_WRITE_AUX` (`0xD4`) and each requiring a `MOUSE_ACK` (`0xFA`) or the whole sequence fails
(`src/init/enable_mouse/enable.rs:26`, `src/init/enable_mouse/mouse_command.rs:21`).

Every controller access is a broker call. Commands go to the status/command port at offset 4, data to the
data port at offset 0, and both writers first spin on the status port until the input-buffer-full bit
clears (`src/constants/ports.rs:16`, `src/init/enable_keyboard/cmd.rs:20`,
`src/init/enable_keyboard/wait_clear.rs:21`). Reads spin until the output-full bit is set before reading
the data port (`src/init/enable_keyboard/read_byte.rs:21`). There is no inline `in`/`out`; the static
gate rejects raw port assembly so every byte crosses the grant checks (`README.md:62`).

Interrupt handling is edge-driven. The pump polls each IRQ grant's sequence with `mk_irq_poll`, drains the
ports, and if either sequence advanced it acknowledges both lines and drains again
(`src/server/pump.rs:24`, `src/server/irq_seq.rs:18`). The second drain after the acknowledge is
deliberate: a byte that lands between the first drain and the IO-APIC unmask raises its edge while the
line is still masked, and a masked edge is dropped rather than latched, so the extra sweep keeps that
byte from holding the line high with no further edge to wake the driver (`src/server/pump.rs:42`). Each
drain reads the status port, counts parity and timeout errors, reads the data byte, and routes it to the
mouse parser when the AUX-data status bit is set or to the keyboard absorber otherwise, up to 16 bytes per
call (`src/poll/drain.rs:24`).

## Protocol and IPC

The driver is a server, not a client of other services. It receives on socket 0 with a 1 ms idle timeout
and, when nothing arrives, waits on the input IRQ with `mk_irq_wait` before looping, so it wakes on an
interrupt rather than busy-spinning (`src/server/runner.rs:51`). Received requests are decoded, validated,
and dispatched to the handlers; replies are sent with `mk_ipc_send` to the kernel reply endpoint
(`src/server/runner.rs:57`).

The syscalls it uses fall into three groups.

Broker device and port-IO, defined in the userland libc (`userland/libc/src/broker/`):

```
  mk_device_list    list ACPI platform records          device.rs:24
  mk_device_claim   bind a device to this pid            device.rs:29
  mk_device_release release a claimed device             (rollback paths)
  mk_pio_grant      mint the i8042 PIO grant (BAR 0)     pio.rs:26
  mk_pio_read       read a port through the grant        pio.rs:40
  mk_pio_write      write a port through the grant       pio.rs:50
  mk_pio_release    release the PIO grant on rollback    (setup/irq.rs)
```

Broker interrupts (`userland/libc/src/broker/irq.rs`): `mk_irq_bind` binds IRQ1 and IRQ12
(`irq.rs:37`), `mk_irq_ack` reopens the line after a drain (`irq.rs:57`), `mk_irq_poll` reads the delivery
sequence (`irq.rs:62`), and `mk_irq_wait` blocks the server until an edge arrives (`irq.rs:72`).

Input posting: `mk_input_event_post` issues syscall `N_MK_INPUT_EVENT_POST` with a pointer to a fixed
`InputEvent` (`userland/libc/src/surface_registry/input_post.rs:22`). On the kernel side the syscall is
gated on `InputSource`: `MkInputEventPost` requires `can_input_source`, which is granted by the
`InputSource`, `Irq`, or `Admin` capabilities (`src/syscall/contract/cap_table/mk.rs:78`,
`src/capabilities/token/types.rs:166`). The handler copies the event in and pushes it onto the shared
MPSC input ring, where a single input-router capsule drains it (`src/syscall/dispatch/router/input_ops.rs:53`,
`src/kernel_core/surface_registry/input_ring.rs:55`). IPC transport is `mk_ipc_recv` for requests and
`mk_ipc_send` for replies (`src/server/runner.rs:51`, `src/server/handlers/poll.rs:51`).

The rings themselves are bounded and drop-on-full. The keyboard ring holds 256 events and, when full,
overwrites the oldest and counts a drop; the mouse ring holds 128 and drops the newest when full
(`src/constants/ports.rs:32`, `src/ring/push.rs:20`, `src/mouse/ring.rs:38`). A full ring never blocks
drain or interrupt handling.

## Security analysis

This capsule is more privileged than an app because it holds real hardware authority, but that authority
is narrow and fully enumerated. Its mask `0x358019` grants CoreExec, IPC, Memory, DeviceEnum, Driver,
Irq, Pio, and InputSource, and nothing else (`Capsule.mk:17`, `src/hardware/ps2_kbd_capsule/spawn.rs:51`).
It has no `Mmio` and no `Dma`, so it cannot map a device BAR into its address space or receive a
DMA-coherent buffer; its only hardware reach is the i8042 port window through kernel-mediated `in`/`out`,
and only the ports the grant covers. It has no FileSystem, Network, Admin, or Debug bit, so it cannot
touch a file, open a socket, or take an administrative action. Every controller access is a broker call
that the kernel checks against the grant, and the ready marker is the one thing it writes outside its
rings, through `mk_debug`.

What it can do is post input. `InputSource` lets it call `mk_input_event_post`, and it uses that to feed
decoded keystrokes and pointer motion into the kernel input ring. Input is sensitive: it is exactly the
user's keystrokes and mouse movement. The isolation that keeps this safe is that the driver only ever
produces events into a bounded kernel ring; it does not choose who receives them. Routing, focus,
grabs, and lock-screen policy live above it in the input router and the compositor, and the whole class
of events can only be grabbed by a few named trusted capsules
([../../subsystems/input/path.md](../../subsystems/input/path.md)). The driver keeps input only in
bounded in-memory rings and diagnostic counters; it writes nothing to disk and keeps no history across a
process exit (`README.md:189`).

The controller is a single shared resource, which is why one capsule owns both the keyboard and the AUX
records rather than splitting them into two owners of one device (`README.md:6`). Malformed input is
data-plane damage, not a fault: a bad mouse sync byte is counted and dropped, a parity or timeout status
is counted, and a full ring drops deterministically and increments its counter, none of which can crash
the driver or wedge the line (`src/mouse/parser.rs:30`, `src/poll/drain.rs:39`, `src/ring/push.rs:22`).
Bring-up is all-or-nothing per phase: a PIO or IRQ failure rolls back the grants already taken, and a
missing keyboard record fails startup outright, so the capsule never runs holding a partial set of grants
(`src/setup/pio.rs:21`, `src/setup/irq.rs:22`, `README.md:200`). If only the mouse enable fails, the
keyboard path stays live and `OP_CONTROLLER_STATUS` reports `mouse_enabled = 0`
(`src/setup/sequence.rs:40`).

## How to contribute

The source lives at `userland/capsule_driver_ps2_input/`. Bring-up is under `src/setup/` and `src/init/`,
the keyboard tables and posting under `src/keymap/`, the mouse parser and posting under `src/mouse/`, the
byte drain under `src/poll/`, the wire protocol under `src/protocol/`, and the IPC server under
`src/server/`.

Common changes:

- To fix or extend the keymap, edit the Set 1 tables. Printable and named base keys live in
  `src/keymap/set1/{left,right,function}.rs`, the extended block in `src/keymap/set1_e0.rs`, and the
  keycode constants in `src/keymap/set1/keycodes.rs`. If you add a modifier, wire its bit in
  `src/keymap/post.rs:31` as well.
- To change mouse decoding, edit `src/mouse/packet.rs` for the byte layout and sign extension, and
  `src/mouse/post.rs` for how a `MouseEvent` becomes kernel input events. Keep posting in the pump path
  only so events are not delivered twice (`src/mouse/parser.rs:26`).
- To add or change an op, add the constant in `src/protocol/ops.rs`, add a handler under
  `src/server/handlers/`, and add its match arm in `src/server/runner.rs:68`. Reply payload sizes are
  bounded by the constants in `src/protocol/limits.rs`, which also size the transmit buffer in the runner.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_driver_ps2_input/Capsule.mk:20`):

```
  make nonos-mk-driver-ps2-input            build the capsule ELF
  make nonos-mk-driver-ps2-input-sign       produce the id cert, manifest, and attestation trailer
  make nonos-mk-driver-ps2-input-verify     verify the signed artifacts against the trust anchor
  make nonos-mk-check-driver-ps2-input-keys check the per-capsule signing keys exist
```

The rule names come from `nonos-mk-$(1)`, `nonos-mk-$(1)-sign`, `nonos-mk-$(1)-verify`, and
`nonos-mk-check-$(1)-keys` where the slug substitutes for `$(1)` (`nonos-mk/capsule.mk:158`, `:261`,
`:263`, `:184`). The README documents rebuilding with `make -B nonos-mk-driver-ps2-input`
(`README.md:281`). For a running desktop that includes this driver, `make nonos-mk-driver-ps2-input-prod`
builds the driver-only kernel profile with feature `microkernel-driver-ps2-input` (`Makefile:970`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every fallible step returns a `Result` with a static error string and the
server drops damaged input rather than panicking; the release profile is `panic = "abort"`,
`Cargo.toml:27`); modular files, one unit per file, with `mod.rs` used only for re-exports; and the AGPL
header at the top of every source file, matching the header on every existing module. The README also
requires that the static checks in `nonos-ci/run-static-checks.sh` pass, which assert no raw PIO
assembly, rollback of partial grants, the presence of the controller-status op, and the AUX mouse path
wired through IRQ12 (`README.md:285`).

## Debugging

The first thing to confirm is that the driver came up. On a successful bring-up it emits
`[driver_ps2] endpoint driver.ps2_kbd0 ready` through `mk_debug` once the setup sequence completes
(`src/setup/sequence.rs:43`). If that marker is absent the device was never claimed and nothing is
posting; the failure is upstream in discovery or the broker claim, not in the decode path. A missing
keyboard record is reported as `ps2 keyboard not present in device list`
([../../subsystems/input/path.md](../../subsystems/input/path.md), `src/setup/sequence.rs:27`).

Once the driver marker is present, the kernel emits one-shot bench markers on the input path:
`input_post_first` on the first successful post into the ring and `input_drain_first` on the first router
drain (`src/kernel_core/surface_registry/input_ring.rs`, `src/syscall/dispatch/router/input_ops.rs:79`).
`input_post_first` present but `input_drain_first` absent means the driver is decoding and posting but the
router is not draining, so check that the input router capsule was spawned and holds IPC. Neither present
means no driver ever posted.

Failure modes and where to look:

- Dead keyboard, driver marker present. Events reach the ring but not the window, or the keyboard was
  never enabled. Poll `OP_GET_STATE` and watch counter 0 (keyboard events seen): if it stays zero, the
  controller is not delivering bytes, which usually means the config-byte flush regressed and
  `CONFIG_IRQ1` got cleared or the disable bit set (`src/init/enable_keyboard/enable.rs:28`). If it
  climbs but nothing appears on screen, the problem is downstream in the router or the target's
  subscription, not this capsule.
- No mouse. `OP_CONTROLLER_STATUS` reports `mouse_enabled` at offset 20; a zero there means the AUX enable
  sequence did not acknowledge, so the keyboard runs but the mouse never came up
  (`src/server/handlers/controller_status.rs:48`, `src/setup/sequence.rs:40`). A non-zero `mouse_enabled`
  with counter 4 (mouse events seen) staying flat points at IRQ12 delivery or a stuck AUX line.
- Stuck or repeating keys. Keys are posted as raw make/break translations, so a stuck key usually means a
  break code was lost. Watch counter 1 (keyboard events dropped): a full ring overwrites the oldest and
  counts a drop, which can eat a break code if the router falls behind (`src/ring/push.rs:22`). Parity and
  timeout counters (indices 2 and 3) rising alongside indicate line-level corruption, not a decode bug.
- Mouse pointer jumps or desyncs. Counter 6 (mouse packet sync errors) rising means the 3-byte packet
  alignment is being lost; the parser refuses a first byte without the sync bit and counts it rather than
  desynchronising, so a high count points at dropped bytes on the AUX line, not the parser
  (`src/mouse/parser.rs:30`).

## Source map

```
  src/main.rs                          _start: heap init, retry setup, run server
  src/discover.rs                      find_ps2_kbd / find_ps2_aux over ACPI platform records
  src/constants/                       ports, status bits, PnP ids, ring capacity
  src/setup/sequence.rs                the ordered bring-up (claim, pio, irq1, aux, enable, marker)
  src/setup/{claim,pio,irq,setup_aux}.rs   the broker grant steps with rollback
  src/init/enable_keyboard/            i8042 keyboard enable, config-byte flush fix, reset, scanning
  src/init/enable_mouse/               AUX enable, config byte, mouse defaults and reporting
  src/init/flush_output.rs             drain stale controller output
  src/poll/drain.rs                    per-byte drain: parity/timeout counts, AUX vs kbd routing
  src/poll/absorb.rs                   Set 1 scancode absorb: prefixes, break bit, ring push, post
  src/keymap/                          Set 1 tables, E0 table, translate, modifier tracking, post
  src/mouse/                           3-byte packet parser, event, ring, kernel input post
  src/ring/                            bounded keyboard event ring and counters
  src/protocol/                        NKBD header, ops, limits, decode/encode, errno, reply endpoint
  src/server/runner.rs                 recv/dispatch loop with idle IRQ wait
  src/server/pump.rs                   IRQ poll, double drain around the unmask, mouse publish
  src/server/handlers/                 health, poll, mouse, state, controller_status
  Capsule.mk                           slug, handle, ports, capability mask, kernel mirror
  src/hardware/ps2_kbd_capsule/        the kernel-side embed and verified spawn
  userland/libc/src/broker/            the device, pio, and irq syscall wrappers
  userland/libc/src/surface_registry/  mk_input_event_post and the InputEvent layout
  nonos-mk/capsule.mk                  the generated nonos-mk-driver-ps2-input[-sign|-verify] targets
```

Every reference above is verified against those trees.

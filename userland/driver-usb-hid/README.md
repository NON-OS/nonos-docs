# capsule_driver_usb_hid (full reference)

`capsule_driver_usb_hid` is the USB HID class driver in the NONOS tree: it turns USB keyboard and
mouse HID reports into `InputEvent`s and posts them into the kernel input ring. It sits on top of the
xHCI transport capsule (`driver.xhci0`), which owns the controller mechanics; this capsule owns HID
class parsing, boot-report normalization, and the input-post path. It is the USB half of the input
story documented in the [input path](../../subsystems/input/path.md).

It is a signed driver capsule, not a GUI app. The kernel spawns it under service handle
`driver.usb_hid0` on service port 4222 with a reply port on 4223, and its capability mask is
`0x200019` (`userland/capsule_driver_usb_hid/Capsule.mk:15`). The source is
`userland/capsule_driver_usb_hid/`.

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

The capsule has two faces. The first is a live driver: it looks up the xHCI transport, enumerates
connected HID devices through it, drains their interrupt IN endpoints, parses each raw HID report,
and publishes the resulting key and pointer events straight into the kernel input ring with
`mk_input_event_post`. The second is a request/reply service on `driver.usb_hid0`: callers can probe
a raw USB configuration descriptor for HID bindings, feed it boot reports directly, poll normalized
event batches, and read counters. Both faces share the same `State` and the same HID parsers.

The split with `driver.xhci0` is strict. PCI enumeration, MMIO, IRQ, DMA, xHCI command and event
rings, port reset, slot lifecycle, endpoint configuration, and interrupt-transfer scheduling all live
in the xHCI capsule; this capsule reaches all of that only by sending xHCI IPC ops and never touches a
device register (`userland/capsule_driver_usb_hid/README.md:20`). What it owns is the HID class layer:
descriptor classification, boot keyboard and boot mouse report normalization, an absolute tablet path,
bounded event queues, and the translation into the kernel's `InputEvent` shape.

The entry point is `no_std`/`no_main`. `_start` initializes the heap and calls `orchestrator::run`,
which never returns (`userland/capsule_driver_usb_hid/src/main.rs:33`). The six top-level modules are
`descriptors` (USB config descriptor parsing), `hid` (report normalization and the input-post path),
`orchestrator` (discovery and the poll loop), `protocol` (the service wire format), `server` (the
request handlers), `state` (the shared runtime state), and `xhci` (the transport client)
(`src/main.rs:22`).

## Identity

Everything the kernel and the service registry need to name and reach the driver comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-usb-hid` | `Capsule.mk:6` |
| Service handle | `driver.usb_hid0` | `Capsule.mk:7`, `src/userspace/capsule_driver_usb_hid/spawn.rs:31` |
| Namespace | `systems.nonos.driver.usb_hid0` | `Capsule.mk:12` |
| Service endpoint | `service:4222:driver.usb_hid0` | `Capsule.mk:13`, `spawn.rs:32` |
| Reply endpoint | `reply:4223:endpoint.4294967314` | `Capsule.mk:14`, `spawn.rs:33`, `spawn.rs:34` |
| Capability mask | `0x200019` | `Capsule.mk:15`, `spawn.rs:51` |
| Binary name | `driver_usb_hid` | `Capsule.mk:10` |
| Kernel mirror | `src/userspace/capsule_driver_usb_hid` | `Capsule.mk:9` |

The mask `0x200019` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x000001  CoreExec        bit()        1     types.rs:56
  0x000008  IPC             bit()        8     types.rs:59
  0x000010  Memory          bit()       16     types.rs:60
  0x200000  InputSource     bit()  2097152     types.rs:77
  --------
  0x200019  = 1 + 8 + 16 + 2097152
```

The kernel spawn path requests exactly those four capabilities and no others
(`src/userspace/capsule_driver_usb_hid/spawn.rs:51`). The one that matters is `InputSource`: it is
what lets this capsule post into the shared input ring. The syscall gate is explicit, not incidental.
`MkInputEventPost` is admitted only when the token satisfies `can_input_source`
(`src/syscall/contract/cap_table/mk.rs:78`), and `can_input_source` grants only to a holder of
`InputSource`, `Irq`, or `Admin` (`src/capabilities/token/types.rs:166`). This capsule holds
`InputSource` and none of the others, so its authority to post input is granted by that one bit alone.

There is no `Network` (4), no `FileSystem` (64), and, crucially, no `Driver` (65536), `DeviceEnum`
(32768), `Mmio` (131072), `Irq` (262144), `Dma` (524288), or `Pio` (1048576) bit in the mask. The
capsule cannot enumerate PCI devices, claim a controller through the
[hardware broker](../../subsystems/hardware-broker/claim.md), map a register through
[MMIO](../../subsystems/hardware-broker/mmio.md), bind an interrupt through
[IRQ](../../subsystems/hardware-broker/irq.md), or allocate DMA. Every hardware effect it needs is an
IPC call to `driver.xhci0`, which holds those grants.

Note that the capsule's own `README.md` still describes the mask as `0x18` (`IPC | Memory`) and states
the capsule makes no input-post call (`README.md:29`, `README.md:60`). That predates the input-post
path; the shipping `Capsule.mk` and the kernel spawn mirror both request `0x200019` including
`InputSource`, which is the authority the code actually uses.

## Operation reference

The service exposes seven operations, defined once in `src/protocol/ops.rs:17` and dispatched by op
code in `src/server/dispatch.rs:22`:

| Op | Code | Input | Output | Handler |
|---|---|---|---|---|
| `OP_HEALTHCHECK` | `0x0001` | empty | status word | `dispatch.rs:23`, `handlers/health.rs:20` |
| `OP_PROBE_CONFIG` | `0x0002` | raw USB config descriptor (<= 512 B) | binding count then 8-byte binding records | `dispatch.rs:24`, `handlers/probe_config.rs:24` |
| `OP_FEED_KEYBOARD_REPORT` | `0x0003` | 8-byte HID boot keyboard report | status word | `dispatch.rs:25`, `handlers/feed_key.rs:21` |
| `OP_FEED_MOUSE_REPORT` | `0x0004` | 3 or 4-byte HID boot mouse report | status word | `dispatch.rs:26`, `handlers/feed_mouse.rs:21` |
| `OP_POLL_KEYS` | `0x0005` | empty | count then 8-byte key events (<= 16) | `dispatch.rs:27`, `handlers/poll_keys.rs:21` |
| `OP_POLL_MOUSE` | `0x0006` | empty | count then 8-byte mouse events (<= 16) | `dispatch.rs:28`, `handlers/poll_mouse.rs:21` |
| `OP_GET_STATE` | `0x0007` | empty | 48-byte counter block | `dispatch.rs:29`, `handlers/get_state.rs:21` |

An op that carries an unexpected body returns `E_INVAL` (-22); an unknown op with an empty body
returns `E_BAD_OP` (-38) (`src/server/dispatch.rs:30`, `src/protocol/errno.rs:17`). `OP_PROBE_CONFIG`
returns `E_NO_HID` (-61) when the descriptor is well formed but carries no boot HID interface
(`handlers/probe_config.rs:37`). `OP_GET_STATE` reports, in order, the descriptor probe count, key
report count, mouse report count, keyboard and mouse queue depths, and keyboard and mouse post-failure
counts (`handlers/get_state.rs:23`).

The service ops are a diagnostic and offline-feed surface. The event stream that actually reaches the
desktop does not go through `OP_POLL_KEYS` or `OP_POLL_MOUSE`; it goes through the input-post path
below.

### How a keypress becomes an input event

The live path starts in the poll loop, which drains an interrupt IN endpoint and hands each report to
`feed_report`. A keyboard report goes to `state.keyboard.feed` (`src/orchestrator/poll/feed_report.rs:30`).

1. `Keyboard::feed` takes the 8-byte boot report: byte 0 is the modifier mask, bytes 2..8 are the up
   to six currently-pressed usage codes. It diffs those against the previous frame `self.prev`. A
   usage that is present now and was not present before is a press; a usage that was present and is now
   gone is a release (`src/hid/keyboard/feed.rs:21`). A usage code of 0 or 1 is filtered as not a real
   key (`src/hid/keyboard/is_real_key.rs:17`).
2. Each transition calls `push_key(scancode, pressed)` (`src/hid/keyboard/push_key.rs:22`). On a press
   it toggles Caps Lock if the usage is `0x39`, resolves an ASCII byte from the usage code, the
   modifier mask, and Caps Lock through `keymap::ascii`, and builds a `KeyEvent`
   (`src/hid/keymap.rs:26`). The ASCII table covers letters with shift/caps XOR, shifted digits and
   punctuation, and the control keys (`src/hid/keymap.rs:28`, `src/hid/punctuation.rs:17`).
3. `push_key` calls `post_key::publish(event)` (`src/hid/post_key.rs:32`). This chooses the kind:
   `INPUT_KIND_KEY_DOWN` on a press, `INPUT_KIND_KEY_UP` on a release. It maps navigation usages
   (arrows, Home/End, Delete, Page Up/Down) to private key codes `0xE000..0xE008`, and otherwise
   sends the ASCII byte or, for a non-ASCII key, `0x2000 | scancode`
   (`src/hid/post_key.rs:37`). It packs the modifier mask into shift/ctrl/alt/meta flag bits
   (`src/hid/post_key.rs:60`).
4. `publish` calls `send` (`src/hid/post_wire.rs:19`), which fills an `InputEvent` and calls
   `mk_input_event_post`, returning success when the syscall returns >= 0. That syscall lands in the
   kernel input ring (below). The event is also mirrored into the capsule's bounded local queue for
   `OP_POLL_KEYS`, and a failed post bumps `post_failures` (`src/hid/keyboard/push_key.rs:29`).

The same `Keyboard::feed` runs when a caller drives `OP_FEED_KEYBOARD_REPORT` with an 8-byte body
(`src/server/handlers/feed_key.rs:28`), so the offline feed and the live endpoint drain converge on
one parser and one post path.

### How a pointer report becomes input events

A mouse report goes to `state.mouse.feed` (`src/orchestrator/poll/feed_report.rs:37`).

1. `Mouse::feed` requires at least 3 bytes: byte 0 is the button mask (low 5 bits), byte 1 is signed
   dx, byte 2 is signed dy, and an optional byte 3 is the signed wheel delta
   (`src/hid/mouse.rs:35`). It only enqueues an event if something changed: motion, wheel, or a button
   transition.
2. Each event calls `post_mouse::publish(event, previous_buttons)` (`src/hid/mouse.rs:66`). That posts
   up to three kinds: a non-zero dx/dy sends `INPUT_KIND_POINTER_REL` with `delta_x`/`delta_y`; a
   non-zero wheel sends `INPUT_KIND_WHEEL` with `delta_y`; and each changed button bit sends
   `INPUT_KIND_BUTTON_DOWN` or `INPUT_KIND_BUTTON_UP` with `code = bit + 1`, so left is 1, right is 2,
   middle is 3 (`src/hid/post_mouse.rs:24`). Each of those is a separate `mk_input_event_post`
   through `send` (`src/hid/post_wire.rs:19`).

The absolute tablet path is separate: a Tablet report (5+ bytes) yields an `INPUT_KIND_POINTER_ABS`
with the 16-bit x/y through `send_abs`, an optional wheel, and button transitions
(`src/hid/tablet.rs:32`, `src/hid/post_wire.rs:25`). The tablet path posts only; it keeps no local
queue and is not exposed as a poll op.

Both `OP_FEED_MOUSE_REPORT` and the live endpoint drain run the same `Mouse::feed`
(`src/server/handlers/feed_mouse.rs:26`).

## Architecture and bring-up

The runtime is one loop, not an event system. `orchestrator::run` blocks until the xHCI transport is
resolvable, looking up the `driver.xhci0` service by name and yielding until it appears
(`src/orchestrator/run.rs:20`, `src/xhci/lookup.rs:21`). Once it has the transport port it calls the
poll loop, which owns discovery, the request service, and the endpoint drain.

Discovery runs through the xHCI transport, entirely over IPC. `enumerate` asks the transport for a
port-status snapshot of up to 255 ports and, for each connected port, calls `configure_port`
(`src/orchestrator/enumerate/run.rs:25`). `configure_port` drives the standard USB bring-up as a
sequence of xHCI ops: enable a slot, address the device, read a 64-byte configuration descriptor, then
walk it for HID bindings and issue a `SET_CONFIGURATION` control transfer
(`src/orchestrator/enumerate/configure_port.rs:28`). For each binding it calls `configure_binding`,
which for a boot keyboard or mouse first issues a `SET_PROTOCOL` control transfer to select the boot
protocol (request type `0x21`, request `0x0B`, value 0 = boot), then asks the transport to allocate a
transfer ring for the interrupt IN endpoint and records a `HidEndpoint`
(`src/orchestrator/binding.rs:24`). The boot protocol matters: it is what makes the fixed 8-byte
keyboard and 3-byte mouse report layouts that the parsers above assume valid, without needing a report
descriptor parser.

Descriptor parsing is a bounded variable-length walk. `hid_bindings` validates the configuration
descriptor header, then walks the record list from offset 9 up to `wTotalLength`, rejecting any record
whose length is under 2 or runs past the total (`src/descriptors/parse.rs:24`). It tracks the current
interface and, on each interrupt IN endpoint, pairs it with that interface. `HidBinding::from_pair`
accepts only HID-class interfaces on interrupt IN endpoints: a boot-subclass keyboard becomes
`HidKind::Keyboard`, a boot-subclass mouse becomes `HidKind::Mouse`, and any other HID-class interface
on such an endpoint becomes `HidKind::Tablet` (`src/descriptors/binding.rs:32`). The walk caps at 8
bindings (`src/protocol/limits.rs:21`).

The poll loop is cooperative. Each iteration it services one service request with `pump_once`, then
drains every bound endpoint with `drain_endpoints`, and, if no endpoints are bound and it has been idle
for a rescan interval of 64 idle polls, it re-enumerates in case a device was hot-plugged
(`src/orchestrator/poll/run.rs:34`, `src/orchestrator/poll/constants.rs:18`). The endpoint drain is a
poll, not an interrupt wait: `drain_endpoint` calls the transport's `interrupt_in` op in a loop, and
each successful report is fed to the parser; an `E_AGAIN` (-11) status means no report is pending and
ends the drain (`src/orchestrator/poll/drain_endpoint.rs:30`, `src/xhci/ops/interrupt_in.rs:38`). So
the actual hardware interrupt handling lives in `driver.xhci0`; this capsule polls the transport for
completed reports. The first time any endpoint binds, the loop emits the debug marker
`[USB-HID-ENUM] tablet bound` once (`src/orchestrator/poll/run.rs:36`).

The service side is a single non-blocking receive per iteration. `pump_once` calls `mk_ipc_recv_from`
on the service inbox with a 1 ms timeout, parses the `NUHI` request envelope, and dispatches it; a
timeout or a malformed frame is simply skipped so the loop keeps draining endpoints
(`src/server/pump_once.rs:26`, `src/protocol/decode.rs:19`).

## Protocol and IPC

The capsule speaks two wire protocols: the `NUHI` service protocol it answers on, and the `NXHC`
transport protocol it calls out to `driver.xhci0` with.

Service protocol (`NUHI`, magic `0x4E55_4849`, version 1, 20-byte header)
(`src/protocol/header.rs:17`). A request is `magic | version | op | flags | pad | request_id |
payload_len`, parsed by `parse`, which rejects a wrong magic, wrong version, or a payload length that
does not match the frame (`src/protocol/decode.rs:19`). A reply echoes the header with a fresh payload
length and begins its body with a 4-byte signed status word; `respond::status` sends just the status,
`respond::payload` sends status plus a body and is used by the probe, poll, and state ops
(`src/server/respond.rs:21`). The seven ops are the table above.

Transport protocol (`NXHC`, magic `0x4E58_4843`, version 1) to `driver.xhci0`
(`src/xhci/wire/constants.rs:19`). The client resolves the transport port by name once at startup
(`src/xhci/lookup.rs:19`) and then issues synchronous `mk_ipc_call` requests, one per op, through
`call` (`src/xhci/call.rs:29`). The ops it uses:

```
  OP_PORT_STATUS           0x0003   snapshot connected ports        constants.rs:26
  OP_ENABLE_SLOT           0x0004   enable a device slot            constants.rs:23
  OP_ADDRESS_DEVICE        0x0006   address the device              constants.rs:20
  OP_GET_CONFIG_DESCRIPTOR 0x0008   read the config descriptor      constants.rs:24
  OP_ALLOC_TRANSFER_RING   0x0009   allocate an interrupt ring      constants.rs:21
  OP_CONTROL_TRANSFER      0x000B   SET_PROTOCOL / SET_CONFIGURATION constants.rs:22
  OP_INTERRUPT_IN          0x000E   poll one interrupt IN report    constants.rs:25
```

The input-post path is the third and most sensitive protocol, and it is a syscall, not IPC.
`mk_input_event_post` is the libc wrapper for the `MkInputEventPost` syscall
(`userland/libc/src/surface_registry/input_post.rs:22`). It takes one `InputEvent` by pointer; the
kernel `do_post` reads it out of user memory and calls `post_input`, which pushes it onto the global
MPSC input ring, bumps the sequence, and wakes the parked router
(`src/syscall/dispatch/router/input_ops.rs:53`, `src/kernel_core/surface_registry/input_ring.rs:55`).
The capability required is exactly `InputSource` (or `Irq`/`Admin`), checked at
`src/syscall/contract/cap_table/mk.rs:78`. From there the single `capsule_input_router` drains the
ring and routes each event to the focused consumer, as documented in the
[input path](../../subsystems/input/path.md). The `InputEvent` struct is the shared 32-byte record
whose userland mirror is `userland/libc/src/surface_registry/types.rs:44`, and the `INPUT_KIND_*`
constants this capsule uses are defined there (`types.rs:23`).

## Security analysis

This capsule is a source of synthetic-capable input, so its trust boundary is worth stating plainly. A
keypress can be a password and injected input can drive another capsule, which is why posting into the
ring is capability-gated at all. Its authority is precisely four bits: `CoreExec`, `IPC`, `Memory`,
and `InputSource` (`Capsule.mk:15`, `src/userspace/capsule_driver_usb_hid/spawn.rs:51`).

What it can do. With `InputSource` it can post any `InputEvent` into the shared ring
(`src/syscall/contract/cap_table/mk.rs:78`). With `IPC` it can answer requests on `driver.usb_hid0`
and call the xHCI transport. That is the whole of it.

What it cannot do. It holds no `Driver`, `DeviceEnum`, `Mmio`, `Irq`, `Dma`, or `Pio` bit, so it
cannot claim a USB controller, map a device register, bind an interrupt line, or allocate DMA. Every
hardware effect is an IPC request to `driver.xhci0`, which holds those grants and applies its own
checks; a bug in this capsule's descriptor walk or report parsing cannot escalate past what the
transport already permits, because this capsule never held controller authority. It also holds no
`FileSystem` or `Network` bit, so it cannot persist keystrokes or exfiltrate them: it keeps only
runtime queues and counters, and process exit destroys them (`README.md:73`).

The event stream itself. Posting is strictly gated, but the input ring's drain side is deliberately
thin, and the confidentiality of the stream against a rogue IPC-capable capsule rests on only one
trusted router being spawned, not on a kernel check on the drain side. That boundary is the router's,
not this capsule's, and it is analysed in full in the
[input path security section](../../subsystems/input/path.md). What this capsule contributes to that
boundary is that it holds `InputSource` and nothing broader, so it can only add events, never read the
ring, and never claim the hardware directly.

Input hygiene at the parser. The descriptor walk rejects malformed lengths and oversized payloads
(`src/descriptors/parse.rs:32`, `handlers/probe_config.rs:25`), the keyboard feed filters non-keys and
diffs against the previous frame so a stuck usage does not repeat, and every event queue is bounded to
64 entries (`src/hid/keyboard/constants.rs:17`, `src/hid/mouse.rs:22`), so a flood of reports cannot
grow memory without bound.

## How to contribute

The source lives at `userland/capsule_driver_usb_hid/`. Descriptor parsing is under `src/descriptors/`,
report normalization and the input-post path under `src/hid/`, discovery and the poll loop under
`src/orchestrator/`, the service wire format under `src/protocol/`, the request handlers under
`src/server/`, and the xHCI transport client under `src/xhci/`.

To add a new service op:

1. Add the op code to `src/protocol/ops.rs:17` and re-export it from `src/protocol/mod.rs:33`.
2. Write the handler as one file under `src/server/handlers/`, exposing `pub fn handle(...)` that
   builds its reply through `respond::status` or `respond::payload` and never panics; declare it in
   `src/server/handlers/mod.rs:17`.
3. Wire the op into the match in `src/server/dispatch.rs:22`, gating on an empty body where the op
   takes no input the way the existing poll and state ops do.

To extend the HID parsing or the input mapping, edit the relevant unit under `src/hid/`: the keyboard
diff and keymap live under `src/hid/keyboard/` and `src/hid/keymap.rs`, the mouse under
`src/hid/mouse.rs`, and the actual `mk_input_event_post` call is centralized in `src/hid/post_wire.rs`,
so a change to how events reach the ring belongs there and nowhere else. Keep controller mechanics out
of this capsule; a new xHCI op belongs in `driver.xhci0` and is called through `src/xhci/`.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_driver_usb_hid/Capsule.mk:17`):

```
  make nonos-mk-driver-usb-hid                 build the capsule ELF
  make nonos-mk-driver-usb-hid-sign            produce the id cert, manifest, and attestation trailer
  make nonos-mk-driver-usb-hid-verify          verify the signed artifacts against the trust anchor
  make nonos-mk-check-driver-usb-hid-keys      check the per-capsule signing keys exist
```

For a running kernel that includes the driver, `make nonos-mk-driver-usb-hid-prod` builds the kernel
profile with the `microkernel-driver-usb-hid` feature, staging the proof, xHCI, and USB HID artifacts
together (`Makefile:980`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns errors as a signed status word, never a panic; the
release profile is `panic = "abort"`, `Cargo.toml:26`); modular files, one unit per file, with `mod.rs`
used only for re-exports; and the AGPL header at the top of every source file, matching the header on
every existing module.

## Debugging

The first live marker is enumeration. Once at least one HID endpoint binds, the poll loop emits
`[USB-HID-ENUM] tablet bound` through `mk_debug`, exactly once
(`src/orchestrator/poll/run.rs:36`). If that line never appears, no HID device was enumerated: the
capsule is either still blocked waiting for the `driver.xhci0` service to appear
(`src/orchestrator/run.rs:20`), or enumeration found no connected port with a boot HID interface
(`src/orchestrator/enumerate/run.rs:31`). Because this capsule holds no device authority, an absent
marker points upstream at xHCI discovery or the port state, not at HID parsing.

Kernel-side input markers confirm the post path. The kernel emits the one-shot bench marker
`input_post_first` on the very first successful post into the ring
(`src/kernel_core/surface_registry/input_ring.rs:68`) and `input_drain_first` on the first drain by the
router (`src/syscall/dispatch/router/input_ops.rs:79`). `input_post_first` present but the desktop
still dead means events are entering the ring but the router or focus path is the suspect, covered in
the [input path debugging section](../../subsystems/input/path.md); `input_post_first` absent after
`[USB-HID-ENUM] tablet bound` means the parser is running but every post is failing, which shows up as
a rising post-failure count in `OP_GET_STATE` (`src/server/handlers/get_state.rs:28`).

Failure modes and where to look:

- Device not enumerated. No `[USB-HID-ENUM] tablet bound`. Confirm `driver.xhci0` is live (the lookup
  loop yields forever until it is, `src/xhci/lookup.rs:21`), then that the device presents a
  boot-subclass HID interface on an interrupt IN endpoint, which is the only shape `HidBinding::from_pair`
  accepts (`src/descriptors/binding.rs:32`).
- No input despite enumeration. The endpoint is bound but reports are not arriving or not posting.
  `interrupt_in` returning `E_AGAIN` continuously means the transport has no completed report to hand
  over, which is an xHCI-side issue (`src/xhci/ops/interrupt_in.rs:38`); a non-zero post-failure count
  in `OP_GET_STATE` means the ring is full or the `InputSource` gate is denying the post.
- Wrong keycodes or characters. The usage-to-ASCII mapping is in `src/hid/keymap.rs:26` and the
  punctuation table in `src/hid/punctuation.rs:17`; the modifier decode and the navigation-key mapping
  are in `src/hid/post_key.rs:37`. A shifted or Caps Lock mismatch is the shift/caps XOR in
  `src/hid/keymap.rs:29`. A key that produces no ASCII is sent as `0x2000 | scancode`
  (`src/hid/post_key.rs:54`), so a consumer seeing those high codes is receiving a key the keymap does
  not cover.
- Malformed descriptor rejected. `OP_PROBE_CONFIG` returns `E_INVAL` for a bad header or oversized
  body and `E_NO_HID` for a valid descriptor with no boot HID interface
  (`src/server/handlers/probe_config.rs:31`).

## Source map

```
  src/main.rs                              _start -> orchestrator::run
  src/orchestrator/run.rs                  wait for driver.xhci0, then enter the poll loop
  src/orchestrator/enumerate/             discovery: port scan, slot/address/descriptor, binding
  src/orchestrator/binding.rs              SET_PROTOCOL boot + alloc interrupt ring per binding
  src/orchestrator/poll/run.rs             the cooperative loop: service + drain + rescan
  src/orchestrator/poll/feed_report.rs     route a raw report to keyboard/mouse/tablet
  src/descriptors/parse.rs                 bounded USB config descriptor walk
  src/descriptors/binding.rs               HID-class + interrupt-IN classification
  src/hid/keyboard/                        boot keyboard diff, keymap, event queue
  src/hid/mouse.rs                         boot mouse normalization and event queue
  src/hid/tablet.rs                        absolute pointer path
  src/hid/post_key.rs                      key event -> InputEvent kind/code/flags
  src/hid/post_mouse.rs                    mouse event -> pointer/wheel/button events
  src/hid/post_wire.rs                     the mk_input_event_post call (send / send_abs)
  src/protocol/                            NUHI header, ops, limits, errno, decode/encode
  src/server/dispatch.rs                   op-code dispatch to handlers
  src/server/handlers/                     health, probe_config, feed_*, poll_*, get_state
  src/server/pump_once.rs                  one non-blocking service receive per loop iteration
  src/xhci/                                the driver.xhci0 client (lookup, call, ops, wire)
  Capsule.mk                               slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_driver_usb_hid/    the kernel-side embed and verified spawn
  userland/libc/src/surface_registry/      InputEvent, kinds, mk_input_event_post wrapper
  src/kernel_core/surface_registry/input_ring.rs   the kernel input ring the driver posts into
  src/syscall/contract/cap_table/mk.rs     the InputSource gate on MkInputEventPost
  nonos-mk/capsule.mk                      the generated nonos-mk-driver-usb-hid[-sign|-verify] targets
```

Every reference above is verified against those trees.

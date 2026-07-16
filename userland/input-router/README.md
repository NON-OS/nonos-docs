# capsule_input_router (full reference)

`capsule_input_router` is the single consumer of the kernel input ring and the fan-out point for the
whole desktop. Driver capsules post hardware events into the kernel ring; this capsule drains that
ring, decides where each event belongs, and delivers it to the owning window over IPC. Pointer events
are placed by hit-testing the window manager; keyboard events follow the manager's focus; a trusted few
capsules can take an exclusive grab. It is the userland counterpart to the kernel
[input subsystem](../../subsystems/input/README.md), and the routing decisions here are the userland
half of the [event path](../../subsystems/input/path.md).

The kernel spawns it under service handle `input_router` on service port 4320 with a reply inbox on port
4321, and its capability mask is `0x19` (`userland/capsule_input_router/Capsule.mk:15`). The source is
`userland/capsule_input_router/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [Routing reference](#routing-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The router is a `no_std`/`no_main` capsule whose whole life is one loop. `_start` initializes the heap
and calls `server::run` (`src/main.rs:32`), which never returns. Each iteration drains any pending IPC
requests, drains a batch of input events from the kernel ring, routes each event to its destination,
pushes a cursor update to the compositor if the cursor moved, and then blocks on the ring's sequence
number when there is nothing left to do (`src/server/runner.rs:38`). So the router never spins when idle
and never blocks while events are pending.

Two kinds of traffic reach the router. Inbound IPC on its own service inbox is how consumers register
interest: a capsule subscribes to the event kinds it wants, and a small set of trusted capsules can ask
for an exclusive grab. Inbound input from the kernel ring is the event stream itself, drained with
`mk_input_event_drain` (`src/sources/kernel_ring.rs:28`). The router holds all the desktop routing policy
that the kernel deliberately keeps out of the ring: the cursor position, who is subscribed to what, which
window holds a grab, which window a held button or key belongs to, and which window the pointer is hovering
(`src/state/context/types.rs:19`).

## Identity

Everything the kernel and the service registry need to name and reach the router comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `input-router` | `Capsule.mk:5` |
| Service handle | `input_router` | `Capsule.mk:6`, `src/userspace/capsule_input_router/spawn.rs:31` |
| Namespace | `systems.nonos.input_router` | `Capsule.mk:11` |
| Service endpoint | `service:4320:input_router` | `Capsule.mk:12`, `spawn.rs:32` |
| Reply endpoint | `reply:4321:endpoint.input_router.reply` | `Capsule.mk:13`, `spawn.rs:33`, `spawn.rs:34` |
| Capability mask | `0x19` | `Capsule.mk:15` |
| Binary name | `input_router` | `Capsule.mk:9` |
| Kernel mirror | `src/userspace/capsule_input_router` | `Capsule.mk:16` |

The mask `0x19` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x0001  CoreExec   bit()  1    types.rs:56
  0x0008  IPC        bit()  8    types.rs:59
  0x0010  Memory     bit() 16    types.rs:60
  ------
  0x0019  = 1 + 8 + 16
```

The kernel spawn path requests exactly those three capabilities and no others
(`src/userspace/capsule_input_router/spawn.rs:50`). This is the important line for a capsule that sees
all input: the router does **not** hold `InputSource`. `InputSource` (capability value `2097152`,
`src/capabilities/types.rs:77`) is the post authority, and `MkInputEventPost` is gated on it
(`src/syscall/contract/cap_table/mk.rs:78`); only the signed driver capsules that own the hardware hold
it. The router only drains, and `MkInputEventDrain` / `MkInputEventWait` are gated on `can_ipc`
(`src/syscall/contract/cap_table/mk.rs:79`), which the `IPC` bit satisfies. So the router can read the
ring and speak IPC, but it cannot itself inject a synthetic event into the ring, and it holds no network,
filesystem, graphics, or hardware capability at all.

## Operation reference

The router exposes four opcodes on its service inbox (`src/protocol/ops.rs:17`). `drain_ipc` reads each
request without blocking (`RECV_NOWAIT`), parses the `NIRS` header, and dispatches on the opcode
(`src/server/drain_ipc.rs:31`). A request that fails to parse is skipped; an unknown opcode with an empty
body gets `E_BAD_OP`, and one with a non-empty body gets `E_INVAL` (`src/server/drain_ipc.rs:51`).

| Op | Opcode | What it does | Handler |
|---|---|---|---|
| `OP_HEALTHCHECK` | `0x0001` | reply `0` to a liveness probe (empty body required) | `ops.rs:17`, `handlers/health.rs:20` |
| `OP_SUBSCRIBE` | `0x0002` | record a pid and a kind mask so it receives those event kinds | `ops.rs:18`, `handlers/subscribe.rs:23` |
| `OP_GRAB_REQUEST` | `0x0003` | claim exclusive keyboard or pointer events (trusted callers only) | `ops.rs:19`, `handlers/grab_request.rs:31` |
| `OP_GRAB_RELEASE` | `0x0004` | drop the caller's grabs (empty body required) | `ops.rs:20`, `handlers/grab_release.rs:21` |

### Subscribe

The body is exactly 8 bytes: a `u32` kind mask and a `u32` pad (`src/protocol/limits.rs:23`). The mask is
a bitset over the `INPUT_KIND_*` values, so bit `n` set means the subscriber wants events whose
`kind == n` (`src/protocol/limits.rs:20`). The handler validates the length and reads the mask, then
`upsert`s it into the subscription table (`src/server/handlers/subscribe.rs:24`). `upsert` updates an
existing entry for that pid, or claims a free slot; a mask of `0` removes the entry, and a full table (16
entries) returns `E_NOMEM` (`src/state/subscriptions/upsert.rs:20`, `src/state/subscriptions/types.rs:17`).
A bad length or a short body is `E_INVAL`.

### Grab request

The body is 8 bytes carrying a `u32` kind mask (`src/server/handlers/grab_request.rs:23`). The handler is
gated first: `is_trusted_grabber` resolves each name in `GRABBERS = [app.boot_splash, app.setup_wizard,
app.input_probe]` to a pid and compares it to the sender, and a non-match is `E_ACCES`
(`src/server/handlers/grab_request.rs:25`, `:32`). A zero mask is `E_INVAL` (`:44`). The grab table then
splits the mask into keyboard bits `0b0000_0011` and pointer bits `0b1111_1100` and stores each class
separately, so a mixed request cannot later cross-match; a class already held by a different pid returns
`E_BUSY` (`src/state/grabs/request.rs:19`, `handlers/grab_request.rs:48`). While a grab is held, matching
events bypass focus and hit-testing entirely.

### Grab release

The body must be empty (`src/server/drain_ipc.rs:48`). The handler drops whichever keyboard and pointer
grabs the caller held and replies `0` (`src/state/grabs/release.rs:20`). It is unconditional and does not
check that the caller was ever trusted, because releasing a grab you do not hold is a no-op.

## Routing reference

`route_event` (`src/route/dispatch.rs:28`) decides the destination of every drained event in a fixed
order. This is the heart of the capsule.

```
  route_event(ctx, ev):
      if a grab covers ev.kind:   deliver to the grab holder; forget the pid if the send fails
      elif is_pointer(ev.kind):   route_pointer(ctx, ev)
      elif is_keyboard(ev.kind):  route_keyboard(ctx, ev)
      else:                       fan out to every subscriber whose mask matches ev.kind
```

1. **Grab first.** `grabs.holder_for(ev.kind)` shifts `1 << kind` and tests it against the stored
   keyboard then pointer masks; a hit short-circuits all focus and hit-test logic and the event goes
   straight to the holder (`src/route/dispatch.rs:29`, `src/state/grabs/holder_for.rs:20`). A failed
   delivery forgets that pid.
2. **Pointer versus keyboard.** `is_pointer` matches `POINTER_REL`, `POINTER_ABS`, `WHEEL`,
   `BUTTON_DOWN`, `BUTTON_UP`, and `TOUCH`; `is_keyboard` matches `KEY_DOWN` and `KEY_UP`
   (`src/route/dispatch.rs:61`, `:65`). Each takes its own path.
3. **Everything else broadcasts** to subscribers whose mask matches the kind, and any pid whose delivery
   fails is forgotten afterward (`src/route/dispatch.rs:43`).

### Pointer path

`route_pointer` (`src/route/pointer/route_pointer.rs:33`) is the most involved path and runs its steps in
this order:

```
  route_pointer(ctx, ev):
      refresh_display(ctx)                       // learn the display size from the compositor, once
      (x, y) = ctx.cursor.apply(ev)               // fold motion into the absolute cursor position
      ctx.cursor_dirty = true
      deliver  = mirror_shell_pointer(ctx, ev, x, y)   // the shell always sees pointer motion
      if a press is active:                        // a drag: keep events on the press target
          deliver += route_to_press(ctx, ev, x, y)
          if ev is BUTTON_UP:  ctx.press = None
          return deliver
      if is_motion(ev):  deliver += hover_motion(ctx, ev, x, y)
      if needs_hit_test(ev):                       // BUTTON_DOWN / BUTTON_UP / TOUCH / WHEEL
          match topmost_target(ctx, x, y):         // ask the WM (QUERY_TOPMOST)
              None or shell:  route_to_shell(ctx, ev, x, y)
              target:         if BUTTON_DOWN: latch Press{target}; route_to_window(ev, target)
```

- `refresh_display` fetches the display bounds from the compositor the first time only, so the cursor
  clamps to the real screen (`src/route/pointer/refresh_display.rs:20`).
- `cursor.apply` folds the event into the absolute position. A `POINTER_REL` adds `delta * mult_x2 / 2`,
  where `mult_x2` is the policy mouse sensitivity clamped to `1..4`; an `ABS` or `TOUCH` maps the device's
  `0..0x7FFF` range onto the display, then clamps into `0..max` (`src/state/cursor.rs:43`).
- `mirror_shell_pointer` sends the shell a `POINTER_ABS` at the cursor position whenever the shell
  subscribed to `POINTER_ABS`, so the shell can track the cursor even while another window is focused
  (`src/route/pointer/mirror_shell_pointer.rs:24`).
- A held button is a drag: while `ctx.press` is set, `route_to_press` keeps sending the target its events
  in the window-local frame frozen at press time, until the `BUTTON_UP` clears the press
  (`src/route/pointer/route_to_press.rs:23`, `route_pointer.rs:40`).
- Motion also drives hover (below).
- A button, touch, or wheel needs a hit test. `topmost_target` asks the window manager which window is
  under the cursor and drops a zero owner pid (`src/route/pointer/topmost_target.rs:20`). If the topmost
  is the shell or there is no window, the event goes to the shell (`route_to_shell`, only for button-down
  and touch); otherwise a `BUTTON_DOWN` latches a `Press` on that window and the event is routed there
  (`route_pointer.rs:52`).
- `route_to_window` is where the coordinate frame changes. A `BUTTON_DOWN` or `TOUCH` first raises and
  focuses the window through `wm::route_focus`, then the event is rewritten so `x`/`y` are the
  window-local coordinates from the WM `Target`, a `POINTER_REL` is promoted to `POINTER_ABS` with its
  deltas zeroed, a subscription check gates delivery, and a failed send forgets the pid
  (`src/route/pointer/route_to_window.rs:27`).

Hover: `hover_motion` caches the topmost window rect so pointer motion inside it routes without a WM
round trip per event. It delivers local-coordinate motion while the cursor is inside the cached rect,
sends a leave (local `(-1, -1)`) and clears the cache when the cursor exits, and re-queries the WM only
every fourth motion event (`REQUERY_EVERY = 4`) to throttle the cross-service call
(`src/route/pointer/hover_motion.rs:25`).

### Keyboard path

`route_keyboard` (`src/route/keyboard.rs:25`) routes to the focused window and tracks per-key targets so
a release follows its press:

```
  route_keyboard(ctx, ev):
      if ev is KEY_UP:  pid = key_targets.take(ev.code)  else  fallback_focus
      else:             pid = wm::query_focus  (cached in last_focus_pid)  else  fallback_focus
      if not subscriptions.allows(pid, ev.kind):  drop (count 0)
      delivered = deliver_one(pid, ev)
      if KEY_DOWN and delivered:  key_targets.remember(ev.code, pid)
      if delivered == 0:          forget_pid(pid)
```

- A `KEY_DOWN` asks the window manager for the focused pid and caches it in `last_focus_pid`; if the WM
  has no focus or the query fails, it falls back to the cached focus, then to the shell
  (`src/route/keyboard.rs:37`, `:61`).
- A `KEY_UP` does not query the WM. It looks up whoever received the matching press in `key_targets` (keyed
  by `code`) and sends the release there, so a focus change while a key is held never strands the release
  in the wrong window (`src/route/keyboard.rs:31`, `src/state/key_targets.rs:53`). The table holds up to 16
  held keys; a full table drops the record and the release falls back to current focus
  (`src/state/key_targets.rs:22`, `:42`).
- Before delivering, the router checks the destination is subscribed to this kind; if not, the event is
  dropped and counted as zero delivered (`src/route/keyboard.rs:46`, `src/state/subscriptions/allows.rs:20`).

## Architecture and lifecycle

The capsule is `no_std`/`no_main` with `panic = "abort"` (`src/main.rs:17`, `Cargo.toml`). The six
top-level modules are `clients` (the outbound service clients), `protocol` (the wire formats), `route`
(the routing decisions), `server` (the loop and request handlers), `sources` (the kernel-ring drain), and
`state` (the routing state) (`src/main.rs:22`).

The routing state is one `Context` (`src/state/context/types.rs:19`), constructed once at startup
(`src/state/context/new.rs:21`). It holds the subscription table (16 slots), the grab table (keyboard and
pointer held separately), the per-key target map, the current press and hover, the cursor, the cached peer
ports for the compositor / wm / policy services, the cached shell pid and last focus pid, a monotonic
request id, and the delivered/dropped counters.

The drain side is deliberately thin. `drain_batch` (`src/sources/kernel_ring.rs:27`) calls
`mk_input_event_drain` for up to `MAX_BATCH = 32` events into a stack array and returns the count; the
kernel ring has already normalised the events, so they land in this address space ready to route
(`src/sources/kernel_ring.rs:19`). When a drain returns zero events, the loop parks in
`mk_input_event_wait` with a 20 ms timeout, carrying the last observed sequence forward
(`src/server/runner.rs:66`).

Lifecycle:

1. The kernel spawns the router first in the desktop GUI fleet, before the compositor, so the rest of the
   desktop comes up behind it (`src/userspace/init/spawn_plan/desktop_fleet.rs:38`, `:72`). The spawn is
   idempotent through an `is_alive` guard, verifies the embedded ELF, cert, manifest, and attestation, and
   registers `input_router` on port 4320 (`src/userspace/capsule_input_router/spawn.rs:37`). It also runs
   as part of the input-probe fleet (`src/userspace/init/spawn_plan/input_probe_fleet.rs:24`).
2. On a successful spawn the kernel logs `[INPUT-ROUTER] capsule spawned` and registers the capsule with
   the lifecycle service (`src/userspace/init/capsule_boot/run.rs:29`, prefix from
   `desktop_fleet.rs:78`).
3. `server::run` builds the `Context`, then loops forever: `drain_ipc`, periodic maintenance every 64th
   iteration (`purge_dead` plus a mouse-sensitivity re-read at most every 2 s), `drain_batch`, route each
   event, push the cursor to the compositor if it moved, and `mk_input_event_wait` when the batch was empty
   (`src/server/runner.rs:38`).

## Protocol and IPC

Inbound requests use the `NIRS` frame: magic `0x4E49_5253` ("NIRS"), version 1, a 20-byte header
(`src/protocol/header.rs:17`). `parse` checks the magic and version, reads the op / flags / request id,
and requires the declared payload length to match the received body exactly (`src/protocol/decode.rs:19`).
A reply reuses the same header with a 4-byte status appended (`src/protocol/encode.rs:19`,
`src/server/respond.rs:21`). The status values the router returns are `0` for success, `E_INVAL` (-22),
`E_ACCES` (-13), `E_BAD_OP` (-38) (`src/protocol/errno.rs:17`), plus `E_NOMEM` (-12) from subscribe and
`E_BUSY` (-16) from grab request (`src/server/handlers/subscribe.rs:21`,
`src/server/handlers/grab_request.rs:22`).

Outbound delivery to a consumer uses a distinct envelope so a subscriber cannot mistake a delivery for a
reply. `encode_delivery` writes the `NINP` frame: magic `0x4E49_4E50` ("NINP"), version 1, two zero
bytes, then the 32-byte `InputEvent` field by field, little-endian, for 40 bytes total
(`src/protocol/delivery.rs:23`, `:29`). Delivery is a point-to-point `mk_ipc_send_to_pid`;
`deliver_one` returns 1 on success and 0 on a failed send, and a zero pid is never sent to
(`src/route/deliver.rs:24`).

The router calls three peer services, each through the shared wire client (`NIRS`-shaped 20-byte header,
150 ms call timeout, `src/clients/wire/call.rs:26`):

- Window manager, service `wm`, magic `0x4E57_4D50` (`src/clients/wm/constants.rs:17`). The router calls
  `OP_QUERY_FOCUS 0x000D` for the focused pid on a key-down (`clients/wm/query_focus.rs:20`),
  `OP_QUERY_TOPMOST 0x000B` for the window under a point, which returns an 8-field `Target` (owner pid,
  window id, local x/y, window x/y/w/h) (`clients/wm/query_topmost.rs:21`, `clients/wm/types.rs:17`), and
  `OP_ROUTE_FOCUS 0x000C` to raise and focus a window on a click (`clients/wm/route_focus.rs:21`).
- Compositor, service `compositor`, magic `0x4E43_4D50` (`src/clients/compositor/constants.rs:17`).
  `OP_DISPLAY_INFO 0x0008` learns the display size once (`clients/compositor/display_size.rs:20`), and
  `OP_CURSOR_UPDATE 0x0006` pushes the new cursor position each iteration it moved
  (`clients/compositor/cursor_update.rs:24`).
- Policy, service `policy`. A typed field read (`OP_GET 0x0001`, field `MOUSE_SENSITIVITY 0x0102`, a `u8`)
  re-reads the mouse sensitivity at most every 2 s, with a 150 ms timeout (`src/clients/policy.rs:29`).

Delivery to consumers is the `NINP` frame above; the desktop shell decodes it inversely
(`userland/capsule_desktop_shell/src/server/input.rs`, documented in the
[event path](../../subsystems/input/path.md#delivery-envelope)).

## Security analysis

The router sees every keystroke and every pointer motion on the machine, so it is one of the most
sensitive capsules in userland, and its defence is that its authority is exactly three bits and nothing
more. The mask `0x19` grants only CoreExec, IPC, and Memory (`Capsule.mk:14`,
`src/userspace/capsule_input_router/spawn.rs:50`). There is no `InputSource` bit, so the router cannot
inject a synthetic event back into the shared ring the way a driver can; posting is gated on `InputSource`
and only the signed hardware drivers hold it (`src/syscall/contract/cap_table/mk.rs:78`,
`src/capabilities/types.rs:77`). There is no network, filesystem, graphics, or hardware capability, so the
router cannot open a socket, touch a file, map a device register, or draw to the screen. Everything it
does that leaves the capsule is an IPC call to a service that holds the real authority: the WM decides
focus and geometry, the compositor owns the framebuffer, and policy owns the sensitivity value. A bug in
routing cannot escalate past the right to ask those services a question and to send a subscriber a frame.

Confidentiality between consumers is enforced by construction:

- **Delivery is subscription-scoped and point-to-point.** A capsule receives an event only if it
  subscribed to that kind, checked on both keyboard delivery and window-pointer delivery
  (`src/state/subscriptions/allows.rs:20`, `src/route/pointer/route_to_window.rs:40`), and delivery is a
  directed `mk_ipc_send_to_pid`, never a broadcast (`src/route/deliver.rs:30`).
- **A window never sees another window's events.** A key goes only to the focused pid, its press target,
  or a grab holder; a window pointer event is rewritten to window-local coordinates before it is sent, so
  a consumer cannot infer the global cursor position or another window's geometry from what it receives
  (`src/route/pointer/route_to_window.rs:32`).
- **Grabs are restricted to three named capsules.** The only way to receive a whole class of events is a
  grab, and `is_trusted_grabber` accepts only `app.boot_splash`, `app.setup_wizard`, and `app.input_probe`
  by pid lookup; anyone else is `E_ACCES` (`src/server/handlers/grab_request.rs:25`). A normal application
  cannot monopolise the keyboard or pointer.
- **Dead consumers are dropped.** Every failed delivery calls `forget_pid`, which verifies the pid is
  actually gone with `mk_pid_alive` and then removes its subscriptions, press, hover, grabs, key targets,
  and cached shell / focus references, so a crashed subscriber cannot wedge routing or keep receiving
  another window's input (`src/state/context/forget_pid.rs:20`). A periodic `purge_dead` sweeps the same
  tables independently of delivery failures (`src/state/context/purge_dead.rs:20`).

One boundary is worth stating plainly, and it is the kernel's, not this capsule's: the kernel gates the
drain syscalls only on `can_ipc`, not on being the router, and the ring has a single waiter slot and a
single tail (see the [ring security analysis](../../subsystems/input/ring.md#security-analysis)). Input
confidentiality against a hypothetical rogue IPC-capable capsule therefore rests on only one trusted
router being spawned with this role, not on a kernel identity check on the drainer. That is a property of
the spawn plan, which brings up exactly one `input_router`.

Isolation from other capsules is the kernel's: the router is a CPL 3 user binary that speaks only IPC and
the input syscalls, verified and enrolled at spawn like every other capsule.

## How to contribute

The source lives at `userland/capsule_input_router/`. The routing decisions are under `src/route/`, the
loop and request handlers under `src/server/`, the routing state under `src/state/`, the outbound service
clients under `src/clients/`, the wire formats under `src/protocol/`, and the kernel-ring drain under
`src/sources/`.

To change routing behaviour, edit the relevant path directly:

1. The dispatch order (grab, pointer, keyboard, broadcast) is `src/route/dispatch.rs:28`. Adding a new
   event class or reordering the decision starts here.
2. Pointer behaviour (cursor mapping, drag, hover, hit-test) is the `src/route/pointer/` tree, driven from
   `route_pointer.rs`. Keyboard focus and key-target behaviour is `src/route/keyboard.rs`.
3. A new IPC operation needs an opcode in `src/protocol/ops.rs:17`, a handler under
   `src/server/handlers/`, and a match arm in `src/server/drain_ipc.rs:44`. Keep the handler in its own
   file and reply through `src/server/respond.rs`.
4. New routing state (a table, a cache) goes under `src/state/` as one unit per file, is added to
   `Context` in `src/state/context/types.rs:19` and its initializer in `src/state/context/new.rs:21`, and
   should be cleared in `src/state/context/forget_pid.rs:20` if it is keyed by pid.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:182`,
included through `userland/capsule_input_router/Capsule.mk:18`):

```
  make nonos-mk-input-router               build the capsule ELF
  make nonos-mk-input-router-sign          produce the id cert, manifest, and attestation trailer
  make nonos-mk-input-router-verify        verify the signed manifest against the trust anchor
  make nonos-mk-check-input-router-keys    check the per-capsule signing keys exist
```

The slug is `input-router` (`Capsule.mk:5`), so every target and every `input-router_*` Makefile variable
uses that hyphenated form (`Makefile:628` includes the `Capsule.mk`; `Makefile:575` bakes
`input-router_BIN` into the desktop image). There is no dedicated `-prod` target for the router; it ships
as part of the desktop GUI fleet artifacts (`input-router_ARTIFACTS`, `Makefile:880`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every fallible path returns an `Option`, a `bool`, or a status errno, and the
release profile is `panic = "abort"`, `Cargo.toml`); modular files, one unit per file, with `mod.rs` used
only for re-exports; and the AGPL header at the top of every source file, matching the header on every
existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[INPUT-ROUTER] capsule spawned` (tag `INPUT-ROUTER`, message `capsule spawned`) from the boot log
(`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line means the
capsule never started, usually a signature, manifest, or capability failure; the error path prints an
`[ERROR]` line instead (`src/userspace/init/capsule_boot/run.rs:32`).

The router sits at the middle of the input path, so the two one-shot kernel bench markers bracket it.
`input_post_first` fires when the first driver posts and `input_drain_first` fires when the router first
drains (`src/kernel_core/surface_registry/input_ring.rs:68`,
`src/syscall/dispatch/router/input_ops.rs:79`). `input_post_first` present but `input_drain_first` absent
means events are entering the ring but the router is not draining it, so check that this capsule was
spawned and holds IPC.

Failure modes and where to look:

- Input reaches no window at all. If the driver posted (its own boot marker is present) but nothing gets
  keys, the split is between the ring and the router. Confirm the router spawned, then that the target
  window actually subscribed: an unsubscribed pid is silently skipped (`src/route/keyboard.rs:46`,
  `src/route/pointer/route_to_window.rs:40`). This is the most common cause of a live window that ignores
  input.
- Keys go to the wrong window, or a window sees a stuck key. Key-down routes by the WM's current focus and
  key-up follows the recorded press target (`src/route/keyboard.rs:31`). If a release lands in the wrong
  place, suspect the `key_targets` table overflowing its 16 held-key slots, which drops the record and
  falls the release back to current focus (`src/state/key_targets.rs:42`), or `wm::query_focus` returning
  a stale pid.
- The pointer moves but clicks never reach a window. The cursor updates from `cursor.apply` regardless of
  hit-testing, so a moving cursor with dead clicks points at the hit test: `wm::query_topmost` returning
  the shell or a zero owner pid routes the button to the shell instead of the window
  (`src/route/pointer/topmost_target.rs:22`, `route_pointer.rs:54`).
- A drag jumps or the button-up is lost. A drag is latched by `ctx.press` at button-down and cleared only
  on `BUTTON_UP` (`src/route/pointer/route_pointer.rs:40`). If the press target dies mid-drag, the failed
  send forgets it and clears the press; if the target window moves itself, the press origin is frozen at
  press time by design (`src/state/press.rs`).
- A grab is stuck. A grab is held until its owner releases it or dies. `grab_release` clears it
  (`src/server/handlers/grab_release.rs:21`), and `purge_dead` / `forget_pid` clear it if the holder
  exited (`src/state/grabs/purge_dead.rs:20`, `src/state/context/forget_pid.rs:31`). A grab that never
  clears means the holder is alive and still holding, or `mk_pid_alive` reports it alive.
- Sensitivity or display bounds look wrong. Sensitivity is re-read from the `policy` service at most every
  2 s and clamped to `1..4` (`src/server/runner.rs:44`, `src/state/cursor.rs:47`); the display size is
  learned from the compositor once (`src/route/pointer/refresh_display.rs:20`), so if the compositor was
  not up when the first pointer event arrived, the cursor clamps to the 1023x767 default until the next
  configure (`src/state/cursor.rs:32`).

The router keeps `delivered_count` and `dropped_count` telemetry (`src/state/context/record.rs:20`), but
as written nothing reports them out, so today they are a debugger inspection target rather than a queryable
statistic.

## Source map

```
  src/main.rs                                _start -> heap_init -> server::run
  src/server/runner.rs                       the loop: drain_ipc, maintenance, drain_batch, route, wait
  src/server/drain_ipc.rs                    parse NIRS requests and dispatch on opcode
  src/server/handlers/{subscribe,grab_request,grab_release,health}.rs   the four op handlers
  src/server/respond.rs                      NIRS reply with a status word
  src/sources/kernel_ring.rs                 drain_batch: mk_input_event_drain, MAX_BATCH = 32
  src/route/dispatch.rs                      route_event: grab / pointer / keyboard / broadcast order
  src/route/pointer/route_pointer.rs         the pointer decision order
  src/route/pointer/{route_to_press,route_to_window,route_to_shell,mirror_shell_pointer,hover_motion}.rs
  src/route/pointer/{topmost_target,refresh_display,shell_pid}.rs   the WM/compositor probes
  src/route/keyboard.rs                      focus routing and per-key targets
  src/route/deliver.rs                       encode_delivery + mk_ipc_send_to_pid
  src/state/context/                         Context: tables, ports, cursor, counters, forget/purge
  src/state/{cursor,press,hover,key_targets}.rs   the pointer and key routing state
  src/state/grabs/                           keyboard/pointer grab table (request, holder_for, release)
  src/state/subscriptions/                   the 16-slot subscription table (upsert, allows, match_kind)
  src/protocol/{header,decode,encode,ops,errno,limits,delivery}.rs   NIRS request and NINP delivery wire
  src/clients/wm/                            QUERY_FOCUS / QUERY_TOPMOST / ROUTE_FOCUS
  src/clients/compositor/                    DISPLAY_INFO / CURSOR_UPDATE
  src/clients/{policy,wire}/                 mouse sensitivity, the shared NIRS call helpers
  Capsule.mk                                 slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_input_router/        the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/             the desktop and input-probe fleet spawn entries
  nonos-mk/capsule.mk                        the generated nonos-mk-input-router[-sign|-verify] targets
```

Every reference above is verified against those trees.
</content>
</invoke>

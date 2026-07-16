# capsule_wm (full reference)

`capsule_wm` is the window manager for the NONOS desktop. It owns the window model, placement, z-order,
focus, and the lifecycle notifications the shell relies on. It does not own pixels: an app registers its
own surface with the compositor and shares the handle directly, and it tells the wm only its
`(window_id, geometry, kind)` so the wm can answer authoritative questions like which window owns the
point under the cursor and who currently holds focus. The [input router](input-router.md) asks it those
questions to route a click; the [compositor](compositor.md) receives a `FOCUS_SET` push when focus
changes; the [desktop shell](desktop-shell.md) subscribes for open and close events. This is the
exhaustive reference; the shorter role note lives in the source `README.md`.

The kernel spawns it under service handle `wm` on service port 4330 with a reply port on 4331, and its
capability mask is `0x19` (`userland/capsule_wm/Capsule.mk:19`). The source is `userland/capsule_wm/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [The window-management actions a user causes](#the-window-management-actions-a-user-causes)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The wm is a `no_std`/`no_main` capsule that is, in steady state, a single request loop over one IPC
inbox. `_start` initializes the heap, blocks in `wait_for_setup` until it can reach the compositor and
read the display geometry, then hands the resulting `Context` to the server loop
(`userland/capsule_wm/src/main.rs:36`, `src/wait_for_setup.rs:19`, `src/setup/run.rs:36`).

It holds a fixed table of up to 256 windows, a single focus reference, a monotonic z counter, and up to
16 lifecycle subscribers (`src/state/context.rs:22`, `src/window/table/types.rs:19`,
`src/state/subscriptions.rs:17`). Apps open, move, resize, focus, raise, minimize, restore, maximize,
and close their windows through it; every one of those verbs is scoped to the sender pid, so a capsule
can only manipulate windows it owns. The two query operations, topmost-hit and current-focus, are the
routing backbone the input router calls, and one privileged operation, `ROUTE_FOCUS`, lets only the
input router move focus on a click. A periodic sweep purges the windows and subscribers of processes
that have exited, so the fixed tables do not fill with the debris of crashed apps
(`src/server/runner/sweep_dead.rs:21`).

The wm carries metadata, not framebuffers. The only pixels it influences are indirect: when focus
changes it pushes a `FOCUS_SET` to the compositor so the compositor can restyle the focused window's
chrome (`src/compositor_client/focus_set.rs:22`).

## Identity

Everything the kernel and the service registry need to name and reach the wm comes from its `Capsule.mk`
and its kernel-side spawn record. The two agree exactly.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `wm` | `Capsule.mk:7` |
| Service handle | `wm` | `Capsule.mk:8`, `src/userspace/capsule_wm/spawn.rs:28` |
| Namespace | `systems.nonos.wm` | `Capsule.mk:13` |
| Service endpoint | `service:4330:wm` | `Capsule.mk:14`, `spawn.rs:29` |
| Reply endpoint | `reply:4331:endpoint.wm.reply` | `Capsule.mk:15`, `spawn.rs:30`, `spawn.rs:31` |
| Capability mask | `0x19` | `Capsule.mk:19`, `spawn.rs:47` |
| Binary name | `wm` | `Capsule.mk:11` |
| Kernel mirror | `src/userspace/capsule_wm` | `Capsule.mk:20` |

The mask `0x19` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x0001  CoreExec   bit()  1    types.rs:56
  0x0008  IPC        bit()  8    types.rs:59
  0x0010  Memory     bit() 16    types.rs:60
  ------
  0x0019  = 1 + 8 + 16
```

The kernel spawn path requests exactly those three capabilities and no others: `Capability::CoreExec.bit()
| Capability::IPC.bit() | Capability::Memory.bit()` (`src/userspace/capsule_wm/spawn.rs:47`). There is no
`Debug` bit (256), no `Network`, no `FileSystem`, no graphics capability, and nothing from the driver
family. The `Capsule.mk` comment is explicit that Debug is deliberately absent because the wm emits no
serial markers in steady state (`Capsule.mk:17`). Note that the source `README.md` still records the mask
as `0x119` (with Debug); that is stale, and `Capsule.mk` plus the kernel spawn are the authoritative
`0x19`.

## Operation reference

The request frame is `NWMP` (magic `0x4E57_4D50`, version 1), a 20-byte header followed by a payload
capped at 256 bytes; `parse` rejects any frame whose declared payload length does not match the buffer
(`src/protocol/header.rs:17`, `src/protocol/limits.rs:17`, `src/protocol/decode.rs:19`). The loop reads
one request, parses it, and dispatches on the opcode; every path replies with at least a 4-byte status
word (`src/server/runner/run.rs:28`, `src/server/runner/dispatch.rs:26`, `src/server/respond.rs:21`).

The fourteen opcodes (`src/protocol/ops.rs:17`):

| Op | Code | Body length | Handler |
|---|---|---|---|
| `OP_HEALTHCHECK` | 0x01 | 0 | `handlers/health.rs:20` |
| `OP_WINDOW_OPEN` | 0x02 | 24 | `handlers/window_open/handle.rs:26` |
| `OP_WINDOW_CLOSE` | 0x03 | 8 | `handlers/window_close.rs:22` |
| `OP_WINDOW_MOVE` | 0x04 | 16 | `handlers/window_move.rs:22` |
| `OP_WINDOW_RESIZE` | 0x05 | 16 | `handlers/window_resize/handle.rs:24` |
| `OP_WINDOW_FOCUS` | 0x06 | 8 | `handlers/window_focus.rs:22` |
| `OP_WINDOW_RAISE` | 0x07 | 8 | `handlers/window_raise.rs:21` |
| `OP_LIFECYCLE_SUBSCRIBE` | 0x08 | 0 | `handlers/lifecycle_subscribe.rs:21` |
| `OP_WINDOW_MINIMIZE` | 0x09 | 8 | `handlers/window_minimize.rs:23` |
| `OP_WINDOW_RESTORE` | 0x0A | 8 | `handlers/window_restore.rs:22` |
| `OP_QUERY_TOPMOST` | 0x0B | 8 | `handlers/query_topmost.rs:27` |
| `OP_ROUTE_FOCUS` | 0x0C | 8 | `handlers/route_focus/handle.rs:24` |
| `OP_QUERY_FOCUS` | 0x0D | 0 | `handlers/query_focus.rs:24` |
| `OP_WINDOW_MAXIMIZE` | 0x0E | 24 | `handlers/window_maximize.rs:22` |

The body lengths are the `WINDOW_*_REQ_LEN` constants in `src/protocol/limits.rs:21`. Each verb handler
checks its exact length first and replies `E_INVAL` (-22) on a mismatch. An opcode the dispatch does not
recognise replies `E_BAD_OP` (-38) when its body is empty and `E_INVAL` otherwise
(`src/server/runner/dispatch.rs:52`); the error codes are defined in `src/protocol/errno.rs:17`.

`OP_WINDOW_OPEN` places the window and wires it into focus and the subscribers
(`src/server/handlers/window_open/handle.rs:26`):

```
  window_open(ctx, sender_pid, req):
      decode (window_id, kind, requested_rect)   // 24-byte body, clamped to display
      if the window already exists for this pid:
          refocus if Normal, reply the current rect (idempotent)
      rect = place(ctx, kind, requested)          // clamp, then collide-and-step
      z    = z.allocate()                          // next monotonic z
      insert Window { owner_pid, window_id, rect, kind, Visible, z }
      if kind == Normal: focus_new_window(...)     // set focus + push FOCUS_SET
      notify_fanout(OPENED, ...)                   // tell subscribers
      reply (status, rect)                         // 20-byte rect reply
```

The reply carries the rect the wm actually chose, which may differ from the requested one after clamping
and collision avoidance (`src/server/respond_window_opened.rs:22`). A repeat open of an existing
`(pid, window_id)` is idempotent: it returns the current rectangle and, for a normal window, re-asserts
focus (`window_open/handle.rs:32`).

Placement is a real but modest policy, not a tiling engine (`src/server/handlers/window_open/place.rs:25`).
The requested rect is first clamped inside the display (`src/geometry/constrain.rs:25`). A non-normal
window, or a normal window that does not overlap any visible normal window, is placed as requested. A
normal window that collides is stepped across a grid from (96, 72) in 40-pixel steps until it finds a
free slot; if the grid is exhausted it falls back to a cascade offset by the count of open normal windows
(`window_open/place.rs:32`, `window_open/collides.rs:21`, `window_open/fallback_slot.rs:23`,
`window_open/constants.rs:17`).

Focus, raise, and z-order:

- `OP_WINDOW_FOCUS` focuses one of the sender's own windows if its kind is focusable, pushing `FOCUS_SET`
  to the compositor and updating the focus model; a non-focusable kind is refused with `E_PERM`
  (`window_focus.rs:41`), and an unchanged focus is a no-op that still replies success (`window_focus.rs:47`).
- `OP_WINDOW_RAISE` stamps the window with the next monotonic z so it draws on top, without changing focus
  (`window_raise.rs:30`).
- `OP_ROUTE_FOCUS` is the privileged focus path. It is refused with `E_PERM` unless the sender is the
  live `input_router` service, resolved by name and cached (`route_focus/handle.rs:25`,
  `route_focus/is_input_router.rs:34`). Given `(owner_pid, window_id)` for any capsule it focuses a
  focusable window and pushes `FOCUS_SET`. This is what turns a pointer click into a focus change.

Visibility:

- `OP_WINDOW_MINIMIZE` hides the window; if it was the focused window the wm first clears focus and
  pushes `FOCUS_SET(0)` so the compositor drops the focus styling (`window_minimize.rs:36`).
- `OP_WINDOW_RESTORE` marks the window visible again and stamps it with a fresh z, but does not on its own
  re-take focus (`window_restore.rs:36`).
- `OP_WINDOW_MAXIMIZE` sets the window to the caller-supplied rect (clamped to display) and raises it; its
  24-byte body carries the target rect, so the caller, not the wm, decides the maximized geometry
  (`window_maximize.rs:41`).

Geometry:

- `OP_WINDOW_MOVE` re-origins the window, keeping its size, clamped to the display (`window_move.rs:45`).
- `OP_WINDOW_RESIZE` changes width and height at the current origin; for a normal window it rejects a size
  that would overlap another visible normal window with `E_INVAL`, so a resize cannot be used to force a
  collision (`window_resize/handle.rs:51`, `window_resize/collides.rs:20`).

Queries (called by the input router):

- `OP_QUERY_TOPMOST` hit-tests a point and returns the topmost visible focusable window containing it,
  packing owner pid, window id, the point in the window's local coordinates, and the window rectangle
  (`query_topmost.rs:40`). The hit test iterates the visible focusable windows and keeps the highest z
  that contains the point (`src/focus/hit_test.rs:30`).
- `OP_QUERY_FOCUS` returns the currently focused `(owner_pid, window_id)`, or zeros when nothing is
  focused (`query_focus.rs:24`).

`OP_LIFECYCLE_SUBSCRIBE` records the sender in the 16-entry subscriber list; a full list replies `E_NOMEM`
(`lifecycle_subscribe.rs:21`, `src/state/subscriptions.rs:28`). `OP_HEALTHCHECK` replies status 0
(`health.rs:20`).

## The window-management actions a user causes

A user never talks to the wm directly. Every gesture arrives as an op from another capsule, and the wm is
where the gesture becomes a state change.

| User action | What actually happens | Handler |
|---|---|---|
| An app opens a window | the app sends `OP_WINDOW_OPEN`; the wm places it, gives it the top z, and focuses it if normal | `window_open/handle.rs:26` |
| Drag a titlebar | the app's toolkit tracks the pointer and sends `OP_WINDOW_MOVE` with the new origin | `window_move.rs:22` |
| Drag a resize edge | the toolkit sends `OP_WINDOW_RESIZE` with the new size; a colliding size is refused | `window_resize/handle.rs:24` |
| Click a window to focus it | the input router hit-tests via `OP_QUERY_TOPMOST`, then sends `OP_ROUTE_FOCUS` for that window | `query_topmost.rs:27`, `route_focus/handle.rs:24` |
| Click a window to raise it | the app sends `OP_WINDOW_RAISE`, bumping its z above the others | `window_raise.rs:21` |
| Click the minimize button | the app sends `OP_WINDOW_MINIMIZE`; focus is dropped if it held it | `window_minimize.rs:23` |
| Restore from the dock | the app sends `OP_WINDOW_RESTORE`, making it visible with a fresh z | `window_restore.rs:22` |
| Click the maximize button | the app sends `OP_WINDOW_MAXIMIZE` with the full-screen rect | `window_maximize.rs:22` |
| Click the close button | the app sends `OP_WINDOW_CLOSE`; the wm drops focus if held and tells subscribers | `window_close.rs:22` |
| An app crashes | the periodic sweep finds the dead pid, removes its windows, clears focus, notifies `CLOSED` | `server/runner/sweep_dead.rs:21` |

Closing a window that holds focus clears the focus model and pushes `FOCUS_SET(0)` before it removes the
window, then broadcasts a `CLOSED` notification to the subscribers (`window_close.rs:41`,
`window_close.rs:64`).

## Architecture and lifecycle

The crate splits into eight top-level modules (`src/main.rs:22`): `compositor_client` (the `NCMP` client
to the compositor), `focus` (the focus model and hit test), `geometry` (the rectangle and clamp),
`protocol` (the `NWMP` wire and opcodes), `server` (the loop, dispatch, handlers, notifications),
`setup` and `wait_for_setup` (bring-up), `state` (the `Context` and subscription list), `window` (the
window struct and table), and `z_order` (the monotonic counter).

The window model is a fixed `[Window; 256]` array; each `Window` is `owner_pid`, `window_id`, a `Rect`,
a `Kind`, a `Visibility`, a `z`, and an `in_use` flag (`src/window/window.rs:27`,
`src/window/table/types.rs:19`). The four kinds are `Normal`, `Dialog`, `Tooltip`, and `Popup`; only
`Normal`, `Dialog`, and `Popup` are focusable, so a tooltip is never a hit-test or focus target
(`src/window/kind.rs:19`, `kind.rs:27`). Visibility is `Visible`, `Minimized`, or `Hidden`
(`src/window/window.rs:20`). Lookups (`find`, `find_mut`) and mutations (`insert`, `remove`,
`remove_one_dead`) all key on `(owner_pid, window_id)` through `Window::matches`, which is the mechanism
that scopes every verb to its owner (`src/window/window.rs:53`, `src/window/table/find.rs:22`,
`src/window/table/insert.rs:22`, `src/window/table/remove.rs:22`, `src/window/table/remove_one_dead.rs:22`).

Placement logic lives entirely under `src/server/handlers/window_open/`: `place.rs` is the collide-and-step
search, `collides.rs` is the overlap predicate over visible normal windows, and `fallback_slot.rs` is the
cascade of last resort. Focus and stacking are two small pieces: the focus model is a single
`Option<FocusedRef>` with `set`, `clear`, and `current` (`src/focus/model.rs:23`), and stacking is a
`u32` counter that hands out a strictly increasing z per open, raise, restore, and maximize, wrapping back
to 1 on overflow (`src/z_order/stack.rs:22`). Draw order is `windows()` sorted by z; the topmost hit is the
maximum z among the windows containing the point (`src/focus/hit_test.rs:30`).

Lifecycle:

1. The kernel spawns the capsule at boot through the desktop fleet plan, which verifies the embedded ELF,
   certificate, manifest, and attestation, registers `wm` on port 4330, and logs `[WM] capsule spawned`
   (`src/userspace/init/spawn_plan/desktop_fleet.rs:100`, `src/userspace/capsule_wm/spawn.rs:34`,
   `src/userspace/init/capsule_boot/run.rs:29`).
2. `wait_for_setup` loops until `setup::run` succeeds: it resolves the `compositor` service, probes it,
   and reads the display width and height, yielding between attempts so a not-yet-ready compositor does
   not spin (`src/wait_for_setup.rs:19`, `src/setup/run.rs:36`, `src/setup/discover.rs:21`,
   `src/compositor_client/display_info.rs:27`). The resulting `Context` starts its request-id counter at
   3 because ids 1 and 2 were spent probing the compositor (`src/setup/run.rs:48`).
3. The server loop blocks on the service inbox with a 250 ms receive timeout so it can run the dead sweep
   every fourth wakeup (`src/server/runner/run.rs:28`, `src/server/runner/constants.rs:17`). The input
   path in is that inbox: window verbs from apps, and topmost and route-focus queries from the input
   router.
4. On each tick the sweep purges dead subscribers, then removes one dead window at a time, clearing focus
   and pushing `FOCUS_SET(0)` if the dead window held focus and broadcasting a `CLOSED` notification for
   each (`src/server/runner/sweep_dead.rs:21`, `src/window/table/remove_one_dead.rs:22`).

## Protocol and IPC

Inbound is the `NWMP` request envelope: a 20-byte header (magic, version, op, flags, request id, payload
length) and up to 256 payload bytes (`src/protocol/header.rs:17`, `src/protocol/decode.rs:19`). Replies
reuse the request's op, flags, and request id and carry a 4-byte status followed by any payload
(`src/protocol/encode.rs:19`, `src/server/respond.rs:21`). Two replies carry more than a status:
`OP_WINDOW_OPEN` returns a 16-byte rect (`src/server/respond_window_opened.rs:22`), `OP_QUERY_TOPMOST`
returns a 32-byte hit descriptor (`src/server/handlers/query_topmost.rs:49`), and `OP_QUERY_FOCUS` returns
an 8-byte `(owner_pid, window_id)` (`src/server/handlers/query_focus.rs:32`).

Outbound lifecycle notifications use a separate `NWMV` envelope (magic `0x4E57_4D56`, version 1) so a
subscriber cannot accidentally reply over the request channel; the body carries the event kind (0 opened,
1 closed), owner pid, window id, and the window's x and y (`src/protocol/notify.rs:21`). Notifications are
fire-and-forget sends to each subscriber pid; a send that fails marks the pid stale and it is dropped from
the list (`src/server/notify_fanout.rs:22`).

The wm calls two peer services:

- Compositor, service `compositor`, magic `NCMP` `0x4E43_4D50`, version 1, 20-byte header
  (`src/compositor_client/wire.rs:10`). It uses `OP` 0x0008 to read the display size at setup with a
  250 ms boot timeout (`src/compositor_client/display_info.rs:19`), and `OP` 0x0004 `FOCUS_SET` to push
  the focused pid (0 to clear) with a 16 ms steady-state timeout (`src/compositor_client/focus_set.rs:19`,
  `wire.rs:13`). The reply header is validated for magic, version, op, request id, and payload length
  before the status is read (`src/compositor_client/wire/reply.rs:24`).
- Registry lookups through `mk_service_lookup` resolve the `compositor` port at setup and the
  `input_router` pid for the `ROUTE_FOCUS` gate (`src/setup/discover.rs:21`,
  `src/server/handlers/route_focus/is_input_router.rs:23`).

On the client side, the input router reaches the wm over the same `NWMP` magic and the same opcodes:
`OP_QUERY_TOPMOST` 0x000B and `OP_ROUTE_FOCUS` 0x000C against service `wm`
(`userland/capsule_input_router/src/clients/wm/constants.rs:17`), and the desktop shell subscribes with
`OP_LIFECYCLE_SUBSCRIBE` (`userland/capsule_desktop_shell/src/wm_client/lifecycle_subscribe.rs`).

## Security analysis

The wm holds `0x19`: CoreExec, IPC, and Memory, and nothing else (`Capsule.mk:19`,
`src/userspace/capsule_wm/spawn.rs:47`). It has no display, surface, network, filesystem, or driver
capability, so it cannot read a framebuffer, open a socket, or touch a device. Its whole authority is to
receive IPC, keep a table of window metadata, and make two kinds of outbound call: a `FOCUS_SET` to the
compositor and a lifecycle notification to subscribers.

- Owner-scoped verbs. Open, close, move, resize, focus, raise, minimize, restore, and maximize are all
  looked up with `(sender_pid, window_id)` through `Window::matches`, so a capsule can only act on its own
  windows and a missing match returns `E_NOENT` rather than touching another app's window
  (`src/window/window.rs:53`, `src/server/handlers/window_move.rs:41`).
- Privileged routing is gated to one caller. `ROUTE_FOCUS` can focus any capsule's window, which is
  exactly what routing a click requires, so it is refused unless the sender is the live `input_router`
  service resolved by name; the cached pid is re-verified whenever the sender changes
  (`src/server/handlers/route_focus/handle.rs:25`, `route_focus/is_input_router.rs:34`).
- Queries expose geometry, not content. `QUERY_TOPMOST` and `QUERY_FOCUS` return which window is where and
  who has focus, the information the input router needs to route input; the wm never sees window pixels to
  leak (`src/server/handlers/query_topmost.rs:40`, `src/server/handlers/query_focus.rs:24`).
- Isolation between the windows it manages. Windows are opaque records keyed by owner; the wm never maps or
  reads another capsule's surface, and one app's window operations cannot read or move another's. A
  focusable check keeps a tooltip from stealing focus or a hit (`src/window/kind.rs:27`), and a resize
  that would overlap a peer normal window is rejected outright (`src/server/handlers/window_resize/handle.rs:51`).
- Bounded and self-cleaning. The window table is 256 entries and the subscriber list 16; both are swept of
  dead pids each tick, so a crashed or malicious app cannot leak entries indefinitely
  (`src/window/table/types.rs:19`, `src/state/subscriptions.rs:17`, `src/server/runner/sweep_dead.rs:21`).
- Input validation. Every handler checks its exact body length and rejects a mismatch with `E_INVAL`, the
  parser rejects a frame whose payload length does not match its buffer, and all geometry passes through a
  saturating clamp that never overflows or panics (`src/protocol/decode.rs:33`,
  `src/geometry/constrain.rs:25`).

Honest limits, stated from the code: placement is collision-avoidance plus a cascade, not a tiling or
layout engine (`src/server/handlers/window_open/place.rs:25`); dialog modality is not enforced, so a
`Dialog` does not block input to its parent; and lifecycle notifications are best-effort sends with no
acknowledgement (`src/server/notify_fanout.rs:35`). Drag-and-drop and edge snapping are not implemented in
the wm.

## How to contribute

The source lives at `userland/capsule_wm/`. The request loop and dispatch are under `src/server/runner/`,
the per-op handlers under `src/server/handlers/`, the placement policy under
`src/server/handlers/window_open/`, focus and hit-testing under `src/focus/`, the window model under
`src/window/`, and the wire under `src/protocol/`.

To change placement, edit `src/server/handlers/window_open/place.rs` (the search) and its constants in
`window_open/constants.rs`; the overlap rule is `window_open/collides.rs` and the cascade of last resort is
`window_open/fallback_slot.rs`. To change focus or stacking, edit `src/focus/model.rs` (the focus
reference) and `src/z_order/stack.rs` (the z counter); the topmost rule is `src/focus/hit_test.rs`.

To add an operation:

1. Add the opcode to `src/protocol/ops.rs:17` and its request length to `src/protocol/limits.rs:21`.
2. Write the handler as one file under `src/server/handlers/` (or a subdirectory for a multi-step op like
   `window_open/`), taking `(ctx, sender_pid, req, body, tx)` and replying through `respond::status` or a
   dedicated encoder; check the exact body length first and scope any window lookup to `sender_pid`.
3. Re-export it from `src/server/handlers/mod.rs` and add the match arm in `src/server/runner/dispatch.rs:33`.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_wm/Capsule.mk:22`):

```
  make nonos-mk-wm               build the capsule ELF
  make nonos-mk-wm-sign          produce the id cert, manifest, and attestation trailer
  make nonos-mk-wm-verify        verify the signed artifacts against the trust anchor
  make nonos-mk-check-wm-keys    check the per-capsule signing keys exist
```

The wm is a member of the desktop fleet, so its signed artifacts (`$(wm_ARTIFACTS)`) are pulled into the
desktop GUI production images, for example `nonos-mk-desktop-gui-prod` and `nonos-mk-terminal-only-prod`
(`Makefile:1079`, `Makefile:1173`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns errors as a status word, never a panic; the wire clamp and
z counter are written to never overflow); modular files, one unit per file, with `mod.rs` used only for
re-exports; and the AGPL header at the top of every source file, matching the header on every existing
module.

## Debugging

The wm is deliberately quiet: Debug is absent from its mask and it emits no serial markers of its own in
steady state (`Capsule.mk:17`). The only wm-related boot marker comes from the kernel spawn path, so the
first thing to confirm is that the capsule started.

- Confirm it spawned. On a successful boot the kernel prints `[WM] capsule spawned` from the desktop-fleet
  boot path (`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line
  means the capsule never started, usually a signature, manifest, or capability failure; the error path
  prints an `[ERROR]` line instead (`src/userspace/init/capsule_boot/run.rs:32`).
- No window opens. If an app's `OP_WINDOW_OPEN` never lands, the wm may still be stuck in
  `wait_for_setup` because the compositor is not answering: setup resolves and probes `compositor` and
  reads the display size before the loop starts, and it retries forever on failure
  (`src/wait_for_setup.rs:19`, `src/setup/run.rs:36`). A window that opens off-screen or in an unexpected
  spot is the placement policy: the requested rect was clamped to the display and then stepped away from a
  collision (`src/geometry/constrain.rs:25`, `src/server/handlers/window_open/place.rs:25`). A repeat open
  that seems to do nothing is the idempotent path returning the existing rect
  (`window_open/handle.rs:32`).
- Focus stuck or lost. Focus is a single reference; it is cleared and `FOCUS_SET(0)` pushed when the
  focused window is closed, minimized, or swept as dead (`window_close.rs:41`, `window_minimize.rs:36`,
  `sweep_dead.rs:24`). If a click does not change focus, suspect the `ROUTE_FOCUS` gate: it is refused with
  `E_PERM` unless the sender is the live `input_router`, so a router that failed to register or a stale
  cached pid stops focus routing (`route_focus/handle.rs:25`, `route_focus/is_input_router.rs:34`). If a
  window will not take focus at all, check its kind: a `Tooltip` is not focusable (`src/window/kind.rs:27`).
- A click lands on the wrong window. The router asks `OP_QUERY_TOPMOST`, which returns the highest-z
  visible focusable window containing the point, in that window's local coordinates
  (`src/server/handlers/query_topmost.rs:40`, `src/focus/hit_test.rs:30`). A window that should be on top
  but is not needs an `OP_WINDOW_RAISE` to bump its z (`window_raise.rs:30`).
- A resize is rejected. `OP_WINDOW_RESIZE` refuses a normal-window size that would overlap another visible
  normal window with `E_INVAL`; move the neighbour or resize smaller (`window_resize/handle.rs:51`,
  `window_resize/collides.rs:20`).
- Windows of a crashed app linger. They are cleared on the next sweep tick, which runs every fourth loop
  wakeup (each wakeup at most 250 ms), not instantly (`src/server/runner/constants.rs:17`,
  `src/server/runner/sweep_dead.rs:21`).

## Source map

```
  userland/capsule_wm/src/main.rs                       _start -> wait_for_setup -> server::run
  userland/capsule_wm/src/wait_for_setup.rs             retry setup until the compositor answers
  userland/capsule_wm/src/setup/                        resolve + probe compositor, read display size
  userland/capsule_wm/src/server/runner/                run loop, dispatch, dead sweep, constants
  userland/capsule_wm/src/server/handlers/              one file per op (window_open/ is multi-step)
  userland/capsule_wm/src/server/handlers/window_open/  decode, place, collides, fallback, focus_new
  userland/capsule_wm/src/server/{respond,respond_window_opened,notify_fanout}.rs  reply + notify
  userland/capsule_wm/src/focus/{model.rs, hit_test.rs} the focus reference and topmost_hit_at
  userland/capsule_wm/src/window/{window.rs, kind.rs}   the Window record and Kind/Visibility
  userland/capsule_wm/src/window/table/                 the 256-entry table (find/insert/remove/sweep)
  userland/capsule_wm/src/z_order/stack.rs              the monotonic z counter
  userland/capsule_wm/src/state/{context.rs, subscriptions.rs}  Context and the 16-entry subscriber list
  userland/capsule_wm/src/geometry/{rect.rs, constrain.rs}      Rect, overlaps/contains, clamp_to_display
  userland/capsule_wm/src/protocol/                     NWMP wire, opcodes, limits, errno, NWMV notify
  userland/capsule_wm/src/compositor_client/            NCMP client: display_info, focus_set, wire
  userland/capsule_wm/Capsule.mk                        slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_wm/spawn.rs                     the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/desktop_fleet.rs        the desktop-fleet spawn entry
  src/capabilities/types.rs                             the capability bit table (0x19 decomposition)
  nonos-mk/capsule.mk                                   the generated nonos-mk-wm[-sign|-verify] targets
```

Every reference above is verified against those trees.
</content>

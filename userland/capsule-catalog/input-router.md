# capsule_input_router

`capsule_input_router` is the single consumer of the kernel's input ring and the fan-out point for the
desktop. It drains hardware events, routes each to the right destination, pointer events by hit-testing
the window manager, keyboard events by focus, and delivers them to subscribed capsules, with exclusive
grabs reserved for a trusted few. It is the userland counterpart to the kernel
[input subsystem](../../subsystems/input/README.md). Service `input_router` on port 4320, capability mask
`0x19`. The source is `userland/capsule_input_router/`.

## Contents

- [The server loop](#the-server-loop)
- [The wire protocol](#the-wire-protocol)
- [Subscribe and grab](#subscribe-and-grab)
- [The routing dispatch](#the-routing-dispatch)
- [Pointer routing](#pointer-routing)
- [Keyboard routing](#keyboard-routing)
- [Hover tracking](#hover-tracking)
- [State](#state)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop

`main.rs:32` initializes the heap and runs the loop (`src/server/runner.rs:31`), which interleaves IPC
handling (subscriptions and grabs) with draining the kernel [input ring](../../subsystems/input/ring.md):

```
  run():
      loop:
          drain_ipc(ctx)                            // handle SUBSCRIBE / GRAB requests (RECV_NOWAIT)
          periodically: ctx.purge_dead(); refresh mouse sensitivity from policy (every 2 s)
          n = drain_batch(batch)                     // mk_input_event_drain, up to MAX_BATCH = 32 events
          for ev in batch[..n]:  route_event(ctx, ev)
          if ctx.cursor_dirty:  compositor::cursor_update(...)
          if n == 0:  mk_input_event_wait(last_seq, INPUT_WAIT_MS, &seq)   // block on the ring sequence
```

It drains a batch of up to 32 events, routes each, pushes the cursor to the compositor when it moved, and
blocks on the ring's sequence number when the ring is empty, so it does not spin.

## The wire protocol

The request frame is `NIRS` (magic `0x4E49_5253`), version 1, a 20-byte header. It delivers events to
subscribers in the `NINP` format (magic `0x4E49_4E50`): an 8-byte header plus an `InputEvent` (kind,
flags, code, x, y, delta_x, delta_y, timestamp_ns).

## Subscribe and grab

Two operations register interest (`src/protocol/ops.rs`): `SUBSCRIBE` (0x2) records a pid and a kind mask
so it receives events of those kinds (`src/server/handlers/subscribe.rs:23`, up to 16 subscribers), and
`GRAB_REQUEST` (0x3) claims exclusive access to keyboard or pointer events. The grab is gated
(`src/server/handlers/grab_request.rs:31`):

```
  is_trusted_grabber(sender):  GRABBERS = [ app.boot_splash, app.setup_wizard, app.input_probe ]
  grab_request:
      if not is_trusted_grabber(sender):  E_ACCES
      if kind_mask == 0:                  E_INVAL
      grabs.request(sender, kind_mask)    // keyboard bits and pointer bits held separately; E_BUSY if held
```

Only the boot splash, the setup wizard, and the input probe, resolved by name to pid, may take an
exclusive grab, so a normal application cannot monopolize the keyboard or pointer; an untrusted grab is
`E_ACCES` and an already-held category is `E_BUSY`.

## The routing dispatch

`route_event` (`src/route/dispatch.rs:28`) decides where each event goes:

```
  route_event(ctx, ev):
      if a grab holds ev.kind:   deliver to the grab holder (exclusive); forget the pid if delivery fails
      elif is_pointer(ev.kind):  route_pointer(ctx, ev)
      elif is_keyboard(ev.kind): route_keyboard(ctx, ev)
      else:                      broadcast to subscribers whose mask matches ev.kind
```

A held grab short-circuits everything for its kind; otherwise pointer and keyboard events take their
specific paths, and other kinds broadcast to matching subscribers. A subscriber whose delivery fails (it
exited) is forgotten.

## Pointer routing

`route_pointer` (`src/route/pointer/route_pointer.rs:33`) is the most involved path:

```
  route_pointer(ctx, ev):
      refresh_display(ctx)                          // re-fetch display bounds from the compositor if stale
      (x, y) = ctx.cursor.apply(ev)                  // apply the motion delta (scaled by policy sensitivity)
      ctx.cursor_dirty = true
      deliver = mirror_shell_pointer(ctx, ev, x, y)  // the shell always sees pointer events
      if a button press is active:                   // a drag: keep events going to the press target
          deliver += route_to_press(ctx, ev, x, y)
          if ev is BUTTON_UP:  ctx.press = None
          return
      if is_motion(ev):  deliver += hover_motion(ctx, ev, x, y)
      if needs_hit_test(ev):                          // button down/up, touch, wheel
          match topmost_target(ctx, x, y):            // ask the WM (QUERY_TOPMOST)
              None or shell:  route_to_shell(ctx, ev, x, y)
              target:         if BUTTON_DOWN: remember Press{target, origin}; route_to_window(ev, target)
```

The cursor position is updated from the motion delta scaled by the policy mouse sensitivity, the shell is
mirrored all pointer events, a drag (a held button) keeps its events flowing to the original press target
until the release, and a fresh click hit-tests the window manager and delivers to the window under the
cursor in local coordinates.

## Keyboard routing

`route_keyboard` (`src/route/keyboard.rs:25`) routes to the focused window and tracks per-key targets so a
release follows its press:

```
  route_keyboard(ctx, ev):
      pid = if KEY_UP:  ctx.key_targets.take(ev.code)  else the WM's current focus (QUERY_FOCUS)
      if not ctx.subscriptions.allows(pid, ev.kind):  drop
      delivered = deliver_one(pid, ev)
      if KEY_DOWN and delivered:  ctx.key_targets.remember(ev.code, pid)   // so KEY_UP goes to the same pid
```

A key-down routes to the window the WM reports focused and remembers which pid received it; the matching
key-up routes to that same pid even if focus has since moved, so a key release is never delivered to the
wrong window.

## Hover tracking

`hover_motion` (`src/route/pointer/hover_motion.rs:27`) tracks which window the pointer is over and sends
enter/leave and local motion, re-querying the window manager only every fourth motion event
(`REQUERY_EVERY`) to throttle the cross-service query; when the pointer leaves a window it sends a leave
event (local coordinates `(-1, -1)`) before clearing the hover.

## State

The `Context` (`src/state/context/types.rs:19`) holds the subscription table (16), the grab table
(keyboard and pointer held separately, `KEYBOARD_BITS`/`POINTER_BITS`), the per-key target map, the current
press and hover, the cursor, the peer ports (compositor, wm, policy), the shell pid, and delivered/dropped
telemetry counters.

## Security analysis

- **Grabs are restricted** to three named system capsules, so an application cannot capture all input.
- **Delivery is subscription-scoped**: a capsule receives an event only if it subscribed to that kind (and
  keyboard delivery additionally checks the subscription allows it).
- **Dead consumers are dropped** on a failed delivery, so a crashed subscriber does not wedge routing.

## Honest gaps

Stated from the code: hover is re-queried on a throttle (every four motion events), so a fast pointer move
can skip a window boundary; key repeat is not implemented (only key-down and key-up); grabs are
keyboard-or-pointer wholesale rather than per-key or per-button; the delivered/dropped telemetry is tracked
but not reported; cursor clipping to the display is not enforced by the router (clients validate absolute
coordinates); and multi-touch is tracked as single-touch.

## Source map

```
  userland/capsule_input_router/src/server/runner.rs, drain_ipc.rs   the loop, ring drain, subscribe/grab
  userland/capsule_input_router/src/server/handlers/{subscribe, grab_request}.rs
  userland/capsule_input_router/src/route/dispatch.rs                route_event
  userland/capsule_input_router/src/route/pointer/{route_pointer, hover_motion}.rs
  userland/capsule_input_router/src/route/keyboard.rs                the focus/key-target routing
  userland/capsule_input_router/src/state/{context/, subscriptions/, grabs/}   the routing state
```

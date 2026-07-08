# capsule_wm

`capsule_wm` is the window manager: it owns window lifecycle, placement, z-order, and focus. Apps open,
move, resize, minimize, maximize, and close windows through it; the [input router](input-router.md)
queries it to hit-test a pointer and to find the focused window; and it purges windows whose owning
process has died. Service `wm` on port 4330, capability mask `0x19`. The source is `userland/capsule_wm/`.

## Contents

- [The server loop](#the-server-loop)
- [The wire protocol](#the-wire-protocol)
- [The operations](#the-operations)
- [Opening a window](#opening-a-window)
- [The queries that drive input](#the-queries-that-drive-input)
- [State](#state)
- [Focus and hit-testing](#focus-and-hit-testing)
- [Lifecycle subscriptions and dead-window sweep](#lifecycle-subscriptions-and-dead-window-sweep)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop

`main.rs:36` initializes the heap, waits for setup (connecting to the compositor), and runs the loop
(`src/server/runner/run.rs:28`) with a periodic dead-window sweep:

```
  run(ctx):
      loop:
          every SWEEP_INTERVAL ticks:  sweep_dead(ctx)      // purge windows of exited pids
          n = mk_ipc_recv_from(inbox, rx, &sender_pid)       // blocking with timeout
          (req, body) = parse(rx[..n])                        // else continue
          dispatch(ctx, sender_pid, req, body, tx)
```

The frame is `NWMP` (magic `0x4E57_4D50`), version 1, a 20-byte header, payload capped at 256 bytes.

## The operations

Fourteen operations (`src/protocol/ops.rs`):

```
  0x1 HEALTHCHECK       0x6 WINDOW_FOCUS       0xB QUERY_TOPMOST
  0x2 WINDOW_OPEN       0x7 WINDOW_RAISE       0xC ROUTE_FOCUS
  0x3 WINDOW_CLOSE      0x8 LIFECYCLE_SUBSCRIBE 0xD QUERY_FOCUS
  0x4 WINDOW_MOVE       0x9 WINDOW_MINIMIZE     0xE WINDOW_MAXIMIZE
  0x5 WINDOW_RESIZE     0xA WINDOW_RESTORE
```

The window verbs, open, close, move, resize, focus, raise, minimize, restore, maximize, are all scoped to
the owner pid, so a capsule can only manipulate its own windows. `QUERY_TOPMOST` and `QUERY_FOCUS` are the
public queries the input router uses.

## Opening a window

`window_open` (`src/server/handlers/window_open/handle.rs:26`) places the window and wires it into focus
and the subscribers:

```
  window_open(ctx, sender_pid, req):
      if the window already exists:  return its current rect (idempotent)
      rect = place(ctx, kind, requested)          // snap to grid, avoid collision
      z    = ctx.z.allocate()                      // monotonic z counter
      insert Window { owner_pid: sender_pid, window_id, rect, kind, Visible, z }
      if kind == Normal:  focus_new_window(ctx, owner_pid, window_id)
      notify_fanout(NOTIFY_KIND_OPENED, owner_pid, window_id, rect)
      return (status, rect)
```

Placement snaps and avoids collision, the new window gets the next monotonic z (so it opens on top), a
normal window takes focus, and the lifecycle subscribers are notified. A repeat open of an existing
window is idempotent and returns the current rectangle.

## The queries that drive input

Two operations exist for the [input router](input-router.md). `QUERY_TOPMOST`
(`src/server/handlers/query_topmost.rs:27`) hit-tests a point and returns the topmost visible focusable
window containing it, packing the owner pid, window id, local coordinates, and window rectangle;
`QUERY_FOCUS` returns the currently focused window. These are the routing backbone: the input router asks
the window manager where a pointer is and who has focus, rather than tracking geometry itself.

## State

The `Context` (`src/state/context.rs:22`) holds the compositor port, the display geometry, and the window
state:

```
  windows: WindowTable      up to MAX_WINDOWS = 256 Windows
  focus:   FocusModel       the currently focused (owner_pid, window_id)
  z:       ZStack           a monotonic z counter
  subscriptions: SubscriptionList   up to 16 lifecycle subscribers

  struct Window { owner_pid, window_id, rect{x,y,w,h}, kind{Normal|Dialog|Splash|Notification},
                  visibility{Visible|Minimized|Hidden}, z, in_use }
```

## Focus and hit-testing

`topmost_hit_at` (`src/focus/hit_test.rs:30`) is the hit-test the topmost query and the router rely on:

```
  topmost_hit_at(windows, px, py):
      best = None
      for w in windows:
          if w.visibility != Visible: skip
          if not w.rect.contains(px, py): skip
          if not w.kind.focusable(): skip
          best = the one with the higher z
      return best as (owner_pid, window_id, local = (px - w.x, py - w.y), win_rect)
```

It iterates the visible focusable windows, keeps the highest z that contains the point, and returns the
owner, the window, and the point in the window's local coordinates, which is what lets a click land on the
right window at the right offset.

## Lifecycle subscriptions and dead-window sweep

The `SubscriptionList` (`src/state/subscriptions.rs:19`) holds up to 16 pids that want lifecycle
notifications; `notify_fanout` sends them a `NOTIFY_KIND_OPENED` or `_CLOSED` when a window opens or
closes. The periodic `sweep_dead` purges windows whose owning pid has exited (and `purge_dead` prunes dead
subscribers), so the fixed 256-window table does not fill with the windows of crashed apps.

## Security analysis

- **Owner-scoped verbs**: a window operation is checked against the sender pid, so a capsule can only move,
  resize, or close its own windows.
- **Public queries only expose geometry**: `QUERY_TOPMOST` and `QUERY_FOCUS` return which window is where
  and who has focus, information the input router needs, not window contents.
- **Bounded tables**: 256 windows and 16 subscribers, swept of dead entries.

## Honest gaps

Stated from the code: `ROUTE_FOCUS` sets the focus model but does not yet push the change to the
compositor's `FOCUS_SET`; placement is basic collision avoidance rather than a full tiling or layout
engine; lifecycle notifications are fire-and-forget to subscribers; and dialog modality is not enforced (a
dialog does not block input to its parent window). Drag-and-drop and window snapping are not implemented.

## Source map

```
  userland/capsule_wm/src/server/runner/{run.rs, dispatch.rs}   the loop, dead sweep, dispatch
  userland/capsule_wm/src/server/handlers/{window_open, query_topmost, window_raise, ...}.rs
  userland/capsule_wm/src/focus/{model.rs, hit_test.rs}          the focus model and topmost_hit_at
  userland/capsule_wm/src/window/{window.rs, table/}            the window table
  userland/capsule_wm/src/state/{context.rs, subscriptions.rs}  context, subscriptions
  userland/capsule_wm/src/z_order/stack.rs                       the monotonic z counter
```

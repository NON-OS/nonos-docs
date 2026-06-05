# GUI Contracts

This page documents the concrete GUI behavior implemented by the userland
desktop stack: boot order, deterministic app window requests, non-overlap
rules, move and resize validation, close teardown, focus, cursor state, and
keyboard and pointer delivery. Read [Desktop](desktop.md),
[Applications](apps.md), and [Capsule Inventory](capsules.md) first.

---

## 1. Runtime shape

The default desktop boot path starts GUI core, then WM, wallpaper catalog,
wallpaper, desktop shell, and desktop services
(`src/userspace/init/spawn_plan/desktop_fleet.rs:17`). GUI core is input router
and compositor (`src/userspace/init/spawn_plan/desktop_fleet.rs:26`). App
capsules are then spawned from the app fleet and app tools plan
(`src/userspace/init/spawn_plan/apps.rs:17`,
`src/userspace/init/spawn_plan/apps_tools.rs:17`).

```
  +----------------+
  | app manifest   |
  | id x y w h     |
  +-------+--------+
          |
  +-------+--------+
  | wm             |
  | place focus    |
  | move resize    |
  +-------+--------+
          |
  +-------+--------+
  | compositor     |
  | scene cursor   |
  +-------+--------+
          |
  +-------+--------+
  | input_router   |
  | key pointer    |
  +----------------+
```

## 2. Window open

Apps using `nonos_app_skeleton` send the manifest window id, kind, requested
initial position, requested width, and requested height to `wm::window_open`
(`userland/app_skeleton/src/setup/announce.rs:33`). The manifest fields are
compiled into each app and include title, window id, kind, initial x and y,
width, height, and input mask (`userland/app_skeleton/src/app/manifest.rs:20`).
The current app manifest values are listed in [Applications](apps.md).

The WM open path clamps the requested rectangle to the display
(`userland/capsule_wm/src/server/handlers/window_open/place.rs:25`). If the
window kind is not normal, or the requested rectangle does not collide, the WM
returns the requested rectangle after clamping
(`userland/capsule_wm/src/server/handlers/window_open/place.rs:27`). For normal
windows that collide, the WM scans candidate slots using placement constants
and returns the first non-colliding rectangle
(`userland/capsule_wm/src/server/handlers/window_open/place.rs:30`). Collision
checks only consider visible normal windows
(`userland/capsule_wm/src/server/handlers/window_open/collides.rs:21`).

## 3. Move and resize

Window move validates request length, window id, x, and y, looks up the
sender-owned window, clamps the candidate rectangle to the display, rejects the
move if a normal window would overlap another visible normal window, then
stores the new rectangle (`userland/capsule_wm/src/server/handlers/window_move.rs:23`,
`userland/capsule_wm/src/server/handlers/window_move.rs:42`,
`userland/capsule_wm/src/server/handlers/window_move.rs:47`,
`userland/capsule_wm/src/server/handlers/window_move.rs:48`,
`userland/capsule_wm/src/server/handlers/window_move.rs:62`).

Window resize validates request length, id, width, and height, rejects zero
width or height, clamps the candidate rectangle to display bounds, rejects a
normal-window collision, and stores the new rectangle on success
(`userland/capsule_wm/src/server/handlers/window_resize/handle.rs:24`,
`userland/capsule_wm/src/server/handlers/window_resize/handle.rs:45`,
`userland/capsule_wm/src/server/handlers/window_resize/handle.rs:49`,
`userland/capsule_wm/src/server/handlers/window_resize/handle.rs:51`,
`userland/capsule_wm/src/server/handlers/window_resize/handle.rs:59`).

The shared resize collision rule ignores the window being changed and rejects
overlap with visible normal windows
(`userland/capsule_wm/src/server/handlers/window_resize/collides.rs:20`).

## 4. Close and teardown

The app skeleton maps a button-down event on the toolkit close button to
`EventOutcome::Close` (`userland/app_skeleton/src/runner/decorations.rs:28`).
When the service frame sees a close result, it calls teardown
(`userland/app_skeleton/src/runner/service_frame.rs:47`). Teardown removes the
compositor scene, clears the input subscription, releases the shared surface,
unmaps the backing memory, sends `window_close` to the WM, and exits
(`userland/app_skeleton/src/runner/teardown.rs:25`).

The WM close handler validates the request, clears compositor focus if the
closed window was focused, removes the window from the table, broadcasts a
closed lifecycle notification, and replies success
(`userland/capsule_wm/src/server/handlers/window_close.rs:22`,
`userland/capsule_wm/src/server/handlers/window_close.rs:41`,
`userland/capsule_wm/src/server/handlers/window_close.rs:58`,
`userland/capsule_wm/src/server/handlers/window_close.rs:64`).

## 5. Focus

Button-down inside an app asks the WM to raise and focus that app window
(`userland/app_skeleton/src/runner/click_focus.rs:20`). The app skeleton also
accepts a desktop-shell control frame for focusing itself, but only if the
sender pid resolves to `desktop_shell`
(`userland/app_skeleton/src/runner/control.rs:28`,
`userland/app_skeleton/src/runner/control.rs:59`). Keyboard routing asks the
WM for the focused pid and falls back to the shell pid when there is no focus
(`userland/capsule_input_router/src/route/keyboard.rs:25`).

## 6. Cursor and pointer delivery

The compositor context owns a cursor tracker alongside scene, damage, focus,
and attach state (`userland/compositor/src/state/context.rs:19`). The input
router context owns subscription state, grab state, cursor state, compositor
port, WM port, shell pid, request id, and delivery counters
(`userland/capsule_input_router/src/state/context/types.rs:19`).

The input router loop drains IPC, periodically purges dead subscribers, drains
a batch from the kernel input ring, routes each event, and waits on the kernel
input sequence when no event is available
(`userland/capsule_input_router/src/server/runner.rs:30`). Pointer routing
refreshes display bounds, applies the event to cursor state, mirrors pointer
events to the shell, asks for the topmost target, and routes to shell or the
target window (`userland/capsule_input_router/src/route/pointer/route_pointer.rs:28`).

Keyboard routing issues a WM focus query, checks that the destination
subscription allows the event kind, delivers one event, forgets a dead target,
and records delivery count (`userland/capsule_input_router/src/route/keyboard.rs:25`).

## 7. App-side delivery contract

The app skeleton parses input deliveries with the `NINP` magic and a fixed
delivery length before converting bytes to an `InputEvent`
(`userland/app_skeleton/src/runner/dispatch.rs:20`). Each service frame
refreshes input subscription, ensures the first frame has been submitted,
drains IPC, handles close, repaints when requested, and waits for display vsync
(`userland/app_skeleton/src/runner/service_frame.rs:28`). This means input,
paint, close, and teardown are part of the shared app runtime, not duplicated
inside each app capsule.


# GUI Contracts

This page documents the concrete GUI behavior implemented by the userland
desktop stack: boot order, deterministic app window requests, non-overlap
rules, on-demand app launch, move and resize validation, close teardown,
focus, cursor state, and keyboard and pointer delivery. Read [Desktop](desktop.md),
[Applications](apps.md), and [Capsule Inventory](capsules.md) first.

---

## 1. Runtime shape

The default desktop boot path starts GUI core, then WM, wallpaper catalog,
wallpaper, desktop shell, and desktop services
(`src/userspace/init/spawn_plan/desktop_fleet.rs:17`). GUI core is input router
and compositor (`src/userspace/init/spawn_plan/desktop_fleet.rs:26`). The init
entry path calls desktop and market after network
(`src/userspace/init/entry.rs:35`, `src/userspace/init/entry.rs:36`). App
fleet source files exist, but the current spawn plan module does not wire them
into `run_init` (`src/userspace/init/spawn_plan/mod.rs:17`,
`src/userspace/init/spawn_plan/mod.rs:41`).
Init registers the kernel-owned `desktop.launcher` service before desktop
startup (`src/userspace/init/entry.rs:34`). The residual init loop drains that
launcher inbox once per loop after the lifecycle tick
(`src/userspace/init/supervisor/loop_impl.rs:25`,
`src/userspace/init/supervisor/loop_impl.rs:29`).

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

## 2. App launch

Desktop launcher entries carry a fixed launch id next to the visible label and
service name (`userland/capsule_desktop_shell/src/state/apps.rs:28`,
`userland/capsule_desktop_shell/src/state/apps.rs:35`). A launcher request
first looks up the app service and sends the app focus control frame if the app
is already alive. If the service lookup fails, it sends an eight-byte launch
frame to `desktop.launcher`
(`userland/capsule_desktop_shell/src/server/handlers/launcher_request/request.rs:19`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_request/launch.rs:19`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_request/launch_frame.rs:17`).

The init launcher registers a kernel-owned inbox and service endpoint named
`desktop.launcher`, using endpoint port `4700` and requiring the IPC
capability (`src/userspace/init/launcher/register.rs:19`). The broker only
accepts messages whose source pid matches the current `desktop_shell` service
owner (`src/userspace/init/launcher/authorize.rs:19`). It decodes only the
`NLAU` versioned eight-byte frame and extracts the launch id from bytes six and
seven (`src/userspace/init/launcher/decode.rs:17`). The allowlist maps ids
`1` through `7` to terminal, file manager, text editor, settings, process
manager, about, and calculator, checking each capsule liveness state before
calling its verified spawn wrapper (`src/userspace/init/launcher/spawn.rs:17`).

```
  +------------------+
  | desktop_shell    |
  | launcher click   |
  +--------+---------+
           |
  +--------+---------+
  | service lookup   |
  | focus if alive   |
  +--------+---------+
           |
  +--------+---------+
  | desktop.launcher |
  | NLAU id          |
  +--------+---------+
           |
  +--------+---------+
  | init broker      |
  | verified spawn   |
  +------------------+
```

## 3. Window open

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

```
+--------------------------+
| app manifest request     |
+------------+-------------+
             |
+------------+-------------+
| wm window open           |
| clamp to display         |
+------------+-------------+
             |
+------------+-------------+
| normal collision check   |
+------------+-------------+
             |
+------------+-------------+
| requested or slot rect   |
| owner focus z state      |
+--------------------------+
```

## 4. Move and resize

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

```
+--------------------------+
| move or resize request   |
+------------+-------------+
             |
+------------+-------------+
| validate sender window   |
| decode geometry          |
+------------+-------------+
             |
+------------+-------------+
| clamp candidate rect     |
+------------+-------------+
             |
+------------+-------------+
| reject normal collision  |
+------------+-------------+
             |
+------------+-------------+
| store rect or fail       |
+--------------------------+
```

## 5. Close and teardown

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

```
+--------------------------+
| close result in app      |
+------------+-------------+
             |
+------------+-------------+
| scene remove             |
| input unsubscribe        |
+------------+-------------+
             |
+------------+-------------+
| release surface backing  |
+------------+-------------+
             |
+------------+-------------+
| wm window close          |
+------------+-------------+
             |
+------------+-------------+
| clear focus if needed    |
| lifecycle notification   |
+--------------------------+
```

## 6. Focus

Button-down inside an app asks the WM to raise and focus that app window
(`userland/app_skeleton/src/runner/click_focus.rs:20`). The app skeleton also
accepts a desktop-shell control frame for focusing itself, but only if the
sender pid resolves to `desktop_shell`
(`userland/app_skeleton/src/runner/control.rs:28`,
`userland/app_skeleton/src/runner/control.rs:59`). Keyboard routing asks the
WM for the focused pid and falls back to the shell pid when there is no focus
(`userland/capsule_input_router/src/route/keyboard.rs:25`).

```
+--------------------------+
| button down or NCTL      |
+------------+-------------+
             |
+------------+-------------+
| wm raise focus           |
+------------+-------------+
             |
+------------+-------------+
| compositor focus state   |
+------------+-------------+
             |
+------------+-------------+
| keyboard route query     |
+------------+-------------+
             |
+------------+-------------+
| focused pid or shell     |
+--------------------------+
```

## 7. Cursor and pointer delivery

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

```
+--------------------------+
| kernel input ring        |
+------------+-------------+
             |
+------------+-------------+
| input router batch       |
+------------+-------------+
             |
+------------+-------------+
| cursor state update      |
| shell mirror             |
+------------+-------------+
             |
+------------+-------------+
| wm topmost or focus      |
+------------+-------------+
             |
+------------+-------------+
| NINP delivery            |
+--------------------------+
```

## 8. App-side delivery contract

The app skeleton parses input deliveries with the `NINP` magic and a fixed
delivery length before converting bytes to an `InputEvent`
(`userland/app_skeleton/src/runner/dispatch.rs:20`). Each service frame
refreshes input subscription, ensures the first frame has been submitted,
drains IPC, handles close, repaints when requested, and waits for display vsync
(`userland/app_skeleton/src/runner/service_frame.rs:28`). This means input,
paint, close, and teardown are part of the shared app runtime, not duplicated
inside each app capsule.

```
+--------------------------+
| NINP frame               |
+------------+-------------+
             |
+------------+-------------+
| parse delivery           |
| normalize event          |
+------------+-------------+
             |
+------------+-------------+
| app on event             |
+------------+-------------+
             |
+------------+-------------+
| repaint close or idle    |
+--------------------------+
```

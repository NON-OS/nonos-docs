# Desktop Service Capsules

This page documents the desktop service capsules below the application layer:
compositor, WM, input router, desktop shell, wallpaper, wallpaper catalog, image
codec, clipboard, login, and toolkit. Read [Desktop](desktop.md),
[GUI Contracts](gui-contracts.md), and [Applications](apps.md) first.

The desktop is not one process. It is a set of small services with separate
state tables and protocol routers. Debug it by following the service that owns
the state you are observing.

---

## 1. Service Split

The compositor owns scenes, damage, focus, cursor, display, and surface attach
state. Its dispatcher accepts healthcheck, scene submit, scene remove, damage
commit, focus set, cursor update, input subscribe, and display info
(`userland/compositor/src/server/runner/dispatch.rs:24`,
`userland/compositor/src/server/runner/dispatch.rs:31`,
`userland/compositor/src/server/runner/dispatch.rs:32`,
`userland/compositor/src/server/runner/dispatch.rs:33`,
`userland/compositor/src/server/runner/dispatch.rs:34`,
`userland/compositor/src/server/runner/dispatch.rs:35`,
`userland/compositor/src/server/runner/dispatch.rs:36`,
`userland/compositor/src/server/runner/dispatch.rs:37`,
`userland/compositor/src/server/runner/dispatch.rs:38`,
`userland/compositor/src/server/runner/dispatch.rs:41`).

The WM owns window lifecycle, geometry, focus, z-order, and lifecycle
subscriptions. Its dispatcher accepts window open, close, move, resize, focus,
raise, minimize, restore, topmost query, focus query, route focus, and lifecycle
subscribe (`userland/capsule_wm/src/server/runner/dispatch.rs:25`,
`userland/capsule_wm/src/server/runner/dispatch.rs:32`,
`userland/capsule_wm/src/server/runner/dispatch.rs:34`,
`userland/capsule_wm/src/server/runner/dispatch.rs:35`,
`userland/capsule_wm/src/server/runner/dispatch.rs:36`,
`userland/capsule_wm/src/server/runner/dispatch.rs:37`,
`userland/capsule_wm/src/server/runner/dispatch.rs:38`,
`userland/capsule_wm/src/server/runner/dispatch.rs:39`,
`userland/capsule_wm/src/server/runner/dispatch.rs:40`,
`userland/capsule_wm/src/server/runner/dispatch.rs:41`,
`userland/capsule_wm/src/server/runner/dispatch.rs:42`,
`userland/capsule_wm/src/server/runner/dispatch.rs:43`,
`userland/capsule_wm/src/server/runner/dispatch.rs:46`,
`userland/capsule_wm/src/server/runner/dispatch.rs:47`).

The input router owns subscriptions, grabs, cursor routing, and delivery. Its
IPC drain path handles healthcheck, subscribe, grab request, and grab release
(`userland/capsule_input_router/src/server/drain_ipc.rs:28`,
`userland/capsule_input_router/src/server/drain_ipc.rs:44`,
`userland/capsule_input_router/src/server/drain_ipc.rs:45`,
`userland/capsule_input_router/src/server/drain_ipc.rs:46`,
`userland/capsule_input_router/src/server/drain_ipc.rs:47`,
`userland/capsule_input_router/src/server/drain_ipc.rs:48`).

```
+--------------------------+
| compositor               |
| pixels scenes damage     |
+------------+-------------+
             |
+------------+-------------+
| wm                       |
| windows focus geometry   |
+------------+-------------+
             |
+------------+-------------+
| input router             |
| subscriptions delivery   |
+------------+-------------+
             |
+------------+-------------+
| desktop shell and apps   |
+--------------------------+
```

## 2. Shell and Session Services

Desktop shell owns ports to compositor, WM, and input router, the input mask,
display geometry, overlay backing, pointer state, tray table, spotlight state,
notification level, and request ids (`userland/capsule_desktop_shell/src/state/context.rs:19`,
`userland/capsule_desktop_shell/src/state/context.rs:20`,
`userland/capsule_desktop_shell/src/state/context.rs:21`,
`userland/capsule_desktop_shell/src/state/context.rs:22`,
`userland/capsule_desktop_shell/src/state/context.rs:23`,
`userland/capsule_desktop_shell/src/state/context.rs:24`,
`userland/capsule_desktop_shell/src/state/context.rs:25`,
`userland/capsule_desktop_shell/src/state/context.rs:26`,
`userland/capsule_desktop_shell/src/state/context.rs:27`,
`userland/capsule_desktop_shell/src/state/context.rs:28`,
`userland/capsule_desktop_shell/src/state/context.rs:29`,
`userland/capsule_desktop_shell/src/state/context.rs:30`,
`userland/capsule_desktop_shell/src/state/context.rs:31`,
`userland/capsule_desktop_shell/src/state/context.rs:32`,
`userland/capsule_desktop_shell/src/state/context.rs:33`,
`userland/capsule_desktop_shell/src/state/context.rs:34`,
`userland/capsule_desktop_shell/src/state/context.rs:35`,
`userland/capsule_desktop_shell/src/state/context.rs:36`). Its dispatcher
handles healthcheck, tray register, tray update, tray remove, notify, and
spotlight open (`userland/capsule_desktop_shell/src/server/dispatch.rs:24`,
`userland/capsule_desktop_shell/src/server/dispatch.rs:25`,
`userland/capsule_desktop_shell/src/server/dispatch.rs:26`,
`userland/capsule_desktop_shell/src/server/dispatch.rs:27`,
`userland/capsule_desktop_shell/src/server/dispatch.rs:28`,
`userland/capsule_desktop_shell/src/server/dispatch.rs:29`,
`userland/capsule_desktop_shell/src/server/dispatch.rs:30`,
`userland/capsule_desktop_shell/src/server/dispatch.rs:31`).

Login owns keyring, desktop shell, and compositor ports, display backing, a
serial, and locked or unlocked session state (`userland/capsule_login/src/state/context/types.rs:16`,
`userland/capsule_login/src/state/context/types.rs:17`,
`userland/capsule_login/src/state/context/types.rs:18`,
`userland/capsule_login/src/state/context/types.rs:19`,
`userland/capsule_login/src/state/context/types.rs:20`,
`userland/capsule_login/src/state/context/types.rs:21`,
`userland/capsule_login/src/state/context/types.rs:22`,
`userland/capsule_login/src/state/context/types.rs:23`,
`userland/capsule_login/src/state/context/types.rs:24`,
`userland/capsule_login/src/state/context/types.rs:25`,
`userland/capsule_login/src/state/context/types.rs:28`,
`userland/capsule_login/src/state/context/types.rs:30`). Its runner handles
healthcheck, start session, end session, and get state
(`userland/capsule_login/src/server/runner.rs:16`,
`userland/capsule_login/src/server/runner.rs:35`,
`userland/capsule_login/src/server/runner.rs:42`,
`userland/capsule_login/src/server/runner.rs:43`,
`userland/capsule_login/src/server/runner.rs:44`,
`userland/capsule_login/src/server/runner.rs:45`,
`userland/capsule_login/src/server/runner.rs:48`).

Clipboard owns a deque of entries, total byte count, max depth, max total bytes,
last activity timestamp, and idle timeout (`userland/capsule_clipboard/src/state/clipboard/types.rs:21`,
`userland/capsule_clipboard/src/state/clipboard/types.rs:22`,
`userland/capsule_clipboard/src/state/clipboard/types.rs:23`,
`userland/capsule_clipboard/src/state/clipboard/types.rs:24`,
`userland/capsule_clipboard/src/state/clipboard/types.rs:25`,
`userland/capsule_clipboard/src/state/clipboard/types.rs:26`,
`userland/capsule_clipboard/src/state/clipboard/types.rs:27`). Its router handles
healthcheck, copy, paste, history list, history get, clear, and idle timeout
(`userland/capsule_clipboard/src/server/handlers/router.rs:25`,
`userland/capsule_clipboard/src/server/handlers/router.rs:30`,
`userland/capsule_clipboard/src/server/handlers/router.rs:31`,
`userland/capsule_clipboard/src/server/handlers/router.rs:32`,
`userland/capsule_clipboard/src/server/handlers/router.rs:33`,
`userland/capsule_clipboard/src/server/handlers/router.rs:34`,
`userland/capsule_clipboard/src/server/handlers/router.rs:35`,
`userland/capsule_clipboard/src/server/handlers/router.rs:36`,
`userland/capsule_clipboard/src/server/handlers/router.rs:37`).

```
+--------------------------+
| desktop shell            |
| tray notify spotlight    |
+------------+-------------+
             |
+------------+-------------+
| login                    |
| locked unlocked session  |
+------------+-------------+
             |
+------------+-------------+
| clipboard                |
| bounded history state    |
+--------------------------+
```

## 3. Wallpaper, Catalog, Image, and Toolkit

Wallpaper owns compositor port, display geometry, backing memory, current ARGB,
alpha, policy, fade timeline, request id, optional policy and catalog ports,
applied wallpaper, and subscriber tick count
(`userland/capsule_wallpaper/src/state/context.rs:19`,
`userland/capsule_wallpaper/src/state/context.rs:20`,
`userland/capsule_wallpaper/src/state/context.rs:21`,
`userland/capsule_wallpaper/src/state/context.rs:22`,
`userland/capsule_wallpaper/src/state/context.rs:23`,
`userland/capsule_wallpaper/src/state/context.rs:24`,
`userland/capsule_wallpaper/src/state/context.rs:25`,
`userland/capsule_wallpaper/src/state/context.rs:26`,
`userland/capsule_wallpaper/src/state/context.rs:27`,
`userland/capsule_wallpaper/src/state/context.rs:28`,
`userland/capsule_wallpaper/src/state/context.rs:29`,
`userland/capsule_wallpaper/src/state/context.rs:30`,
`userland/capsule_wallpaper/src/state/context.rs:31`,
`userland/capsule_wallpaper/src/state/context.rs:32`,
`userland/capsule_wallpaper/src/state/context.rs:33`). Its dispatcher handles
healthcheck, set wallpaper, get wallpaper, set policy, and fade
(`userland/capsule_wallpaper/src/server/runner/dispatch.rs:24`,
`userland/capsule_wallpaper/src/server/runner/dispatch.rs:25`,
`userland/capsule_wallpaper/src/server/runner/dispatch.rs:26`,
`userland/capsule_wallpaper/src/server/runner/dispatch.rs:27`,
`userland/capsule_wallpaper/src/server/runner/dispatch.rs:28`,
`userland/capsule_wallpaper/src/server/runner/dispatch.rs:31`,
`userland/capsule_wallpaper/src/server/runner/dispatch.rs:32`).

Wallpaper catalog polls its endpoint, decodes a fixed header, then handles get
count, get size, get chunk, and get slug (`userland/capsule_wallpaper_catalog/src/server/runner.rs:24`,
`userland/capsule_wallpaper_catalog/src/server/runner.rs:28`,
`userland/capsule_wallpaper_catalog/src/server/runner.rs:36`,
`userland/capsule_wallpaper_catalog/src/server/runner.rs:40`,
`userland/capsule_wallpaper_catalog/src/server/runner.rs:41`,
`userland/capsule_wallpaper_catalog/src/server/runner.rs:42`,
`userland/capsule_wallpaper_catalog/src/server/runner.rs:43`,
`userland/capsule_wallpaper_catalog/src/server/runner.rs:44`).

Image codec blocks on IPC, parses a request, handles healthcheck, and dispatches
PNG, BMP, LZ4 raw, and JPEG decode requests to one decode handler
(`userland/capsule_image_codec/src/server/runner.rs:28`,
`userland/capsule_image_codec/src/server/runner.rs:36`,
`userland/capsule_image_codec/src/server/runner.rs:46`,
`userland/capsule_image_codec/src/server/runner.rs:51`,
`userland/capsule_image_codec/src/server/runner.rs:52`,
`userland/capsule_image_codec/src/server/runner.rs:53`). The decode handler
uses a 16384-pixel scratch buffer, maps the op to PNG, BMP, JPEG, or LZ4 raw
decoding, registers an ARGB surface, and returns handle, dimensions, stride,
format, and byte length (`userland/capsule_image_codec/src/server/handlers/decode.rs:23`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:25`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:26`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:27`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:28`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:29`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:30`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:31`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:36`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:38`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:39`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:40`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:41`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:42`,
`userland/capsule_image_codec/src/server/handlers/decode.rs:43`).

Toolkit owns global atomic theme colors and a revision counter
(`userland/toolkit/src/theme/store/state.rs:18`,
`userland/toolkit/src/theme/store/state.rs:19`,
`userland/toolkit/src/theme/store/state.rs:20`,
`userland/toolkit/src/theme/store/state.rs:21`,
`userland/toolkit/src/theme/store/state.rs:22`,
`userland/toolkit/src/theme/store/state.rs:23`). Its dispatcher handles
healthcheck, theme apply, theme get, animation tick, and component render
(`userland/toolkit/src/server/dispatch.rs:25`,
`userland/toolkit/src/server/dispatch.rs:26`,
`userland/toolkit/src/server/dispatch.rs:27`,
`userland/toolkit/src/server/dispatch.rs:28`,
`userland/toolkit/src/server/dispatch.rs:29`,
`userland/toolkit/src/server/dispatch.rs:30`,
`userland/toolkit/src/server/dispatch.rs:31`).

```
+--------------------------+
| wallpaper service        |
| policy catalog decode    |
+------------+-------------+
             |
+------------+-------------+
| wallpaper catalog        |
| count size chunk slug    |
+------------+-------------+
             |
+------------+-------------+
| image codec              |
| png bmp jpeg lz4         |
+------------+-------------+
             |
+------------+-------------+
| toolkit                  |
| theme animation render   |
+--------------------------+
```

## 4. Failure Map

| Symptom | First source path to inspect | Why |
|---------|------------------------------|-----|
| Surface appears but does not repaint | `userland/compositor/src/server/runner/dispatch.rs:35` | Damage commit is the compositor state mutation that makes later frame pacing useful. |
| Window cannot move or resize | `userland/capsule_wm/src/server/runner/dispatch.rs:36` | WM owns geometry mutation and dispatches move and resize requests. |
| Input subscription has no effect | `userland/capsule_input_router/src/server/drain_ipc.rs:46` | Subscribe is handled by input router, not compositor or WM. |
| Launcher tray or notification state is wrong | `userland/capsule_desktop_shell/src/server/dispatch.rs:27` | Shell owns tray and notify requests. |
| Clipboard history grows incorrectly | `userland/capsule_clipboard/src/state/clipboard/types.rs:21` | Clipboard owns bounded history and total byte counters. |
| Wallpaper does not apply | `userland/capsule_wallpaper/src/server/runner/dispatch.rs:27` | Wallpaper changes enter through set wallpaper, policy, or fade handlers. |
| Decoded image has no surface handle | `userland/capsule_image_codec/src/server/handlers/decode.rs:36` | Decode must register an ARGB surface before replying. |
| Toolkit colors do not update | `userland/toolkit/src/server/dispatch.rs:28` | Theme apply is the toolkit path that mutates theme state. |

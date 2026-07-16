# capsule_desktop_shell (full reference)

`capsule_desktop_shell` is the top-level chrome of the NONOS desktop: the launcher dock at the bottom,
the menu bar and status indicators at the top, the notification toasts, the system tray, and the
spotlight panel. It is the coordination hub that ties the compositor, window manager, input router,
wallpaper, and market together, but it holds no more authority than any other graphics client. This is
the exhaustive reference; the [desktop overview](desktop.md) is the short version.

The kernel spawns it under service handle `desktop_shell` on service port 4410 with a reply inbox on
port 4411, and its capability mask is `0x1819` (`userland/capsule_desktop_shell/Capsule.mk:16`). The
source is `userland/capsule_desktop_shell/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [User reference](#user-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The shell is a `no_std`/`no_main` capsule. `_start` initializes the heap, blocks in `wait_for_setup`
until every required peer is up and a shared overlay surface is registered, then runs the server loop
(`src/main.rs:37`, `src/wait_for_setup.rs:19`, `src/server/runner/run.rs:29`). It supplies its own frame
protocol on its service port and its own paint routines; it is not built on the app skeleton the way the
[terminal](terminal/README.md) is.

The chrome is drawn into one full-screen ARGB8888 overlay that the shell allocates during setup and
submits to the compositor as a scene at z-order 1, above every application window
(`src/setup/prime/register.rs:27`, `src/setup/prime/register.rs:57`). After that submission the shell
paints into the overlay's own backing memory and issues a compositor damage commit to have the changed
rectangle presented, exactly like any other client (`src/render/chrome.rs:32`,
`src/compositor_client/damage_commit.rs:22`). The frame protocol on its inbound port is `NDSH`
(magic `0x4E44_5348`), version 1, with a 20-byte header (`src/protocol/header.rs:17`).

## Identity

Everything the kernel and the service registry need to name and reach the shell comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `desktop-shell` | `Capsule.mk:5` |
| Service handle | `desktop_shell` | `Capsule.mk:6`, `src/userspace/capsule_desktop_shell/spawn.rs:31` |
| Namespace | `systems.nonos.desktop_shell` | `Capsule.mk:11` |
| Service endpoint | `service:4410:desktop_shell` | `Capsule.mk:12`, `spawn.rs:32` |
| Reply endpoint | `reply:4411:endpoint.desktop_shell.reply` | `Capsule.mk:13`, `spawn.rs:33`, `spawn.rs:34` |
| Capability mask | `0x1819` | `Capsule.mk:16` |
| Binary name | `desktop_shell` | `Capsule.mk:9` |
| Kernel mirror | `src/userspace/capsule_desktop_shell` | `Capsule.mk:17` |

The mask `0x1819` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x0001  CoreExec                => 1     types.rs:56
  0x0008  IPC                     => 8     types.rs:59
  0x0010  Memory                  => 16    types.rs:60
  0x0800  GraphicsDisplayQuery    => 2048  types.rs:67
  0x1000  GraphicsSurfaceCreate   => 4096  types.rs:68
  ------
  0x1819  = 1 + 8 + 16 + 2048 + 4096
```

The kernel spawn path requests exactly those five capabilities and no others
(`src/userspace/capsule_desktop_shell/spawn.rs:50`). There is no `Network` bit (4), no `FileSystem` bit
(64), and no hardware, driver, DMA, or `GraphicsPresent` (16384) capability in the mask
(`types.rs:58`, `types.rs:62`, `types.rs:70`). The shell can create a surface, ask the display for its
size, and speak IPC; it cannot present a frame itself, read a device, open a socket, or touch the
filesystem.

## User reference

The shell has no keyboard commands. Every user action is a pointer or touch gesture handled by the input
path (`src/server/input.rs:28`), which decodes the input-router `NINP` frame (magic `0x4E49_4E50`) and
reads the event kind, x, and y (`src/server/input.rs:29`, `src/server/input.rs:35`).

### Launching an app

The dock is the launcher. A button-down or touch inside a dock entry runs `launcher_focus`
(`src/server/input.rs:62`, `src/server/handlers/launcher_focus.rs:24`). It hit-tests the pointer against
each entry's rectangle, and on a hit it sends that app an `NCTL` focus-self control frame through
`launcher_request` (`src/server/handlers/launcher_focus.rs:33`,
`src/server/handlers/launcher_request.rs:26`). The frame is magic `NCTL` version 1, op `FOCUS_SELF` = 1,
sent to the target's pid after a service lookup (`launcher_request.rs:22`, `launcher_request.rs:32`,
`launcher_request.rs:42`). On success the shell marks the entry with a launch pulse for 900 ms and
repaints (`launcher_focus.rs:34`, `src/state/taskbar/mark_launch.rs:19`).

An important honesty point: the dock does not spawn a process. Every desktop app is already running, so
clicking a dock entry only focuses it. If the target service is not registered, the lookup returns no pid
and nothing happens (`launcher_request.rs:36`).

The dock shows a fixed set of nine apps (`src/state/apps.rs:36`), each a label paired with the service
handle it focuses:

| Slot | Label | Service focused | Source |
|---|---|---|---|
| 1 | Terminal | `app.terminal` | `apps.rs:37` |
| 2 | Files | `app.file_manager` | `apps.rs:38` |
| 3 | Editor | `app.text_editor` | `apps.rs:39` |
| 4 | Settings | `app.settings` | `apps.rs:40` |
| 5 | Processes | `app.process_manager` | `apps.rs:41` |
| 6 | About | `app.about` | `apps.rs:46` |
| 7 | Calculator | `app.calculator` | `apps.rs:47` |
| 8 | Snake | `app.snake` | `apps.rs:52` |
| 9 | Wallet | `app.nonos_wallet` | `apps.rs:53` |

Each dock entry is tinted by its state: a green underline and lighter fill when the app is the active
window, a pulsing tint for 900 ms after a launch click, and a dimmer "open" tint while the app has a live
window (`src/render/bottom_taskbar.rs:34`, `bottom_taskbar.rs:54`). The window manager drives the
active/open state through its lifecycle notifications (see below), so the dock reflects what is really on
screen, not just what was clicked.

### Revealing and hiding the dock

The dock auto-hides. When it is hidden, moving the pointer into the bottom 4-pixel band of the screen
reveals it, and a touch or click near the bottom of the screen reveals it too
(`src/server/input.rs:70`, `src/server/input.rs:54`, `src/state/taskbar/reveal.rs:19`). A reveal lasts
1800 ms unless a click keeps it open (`reveal.rs:21`). When the dock is visible and the pointer moves
above it while no entry is open, it collapses again (`src/server/input.rs:81`).

### Status indicators

The menu bar carries a right-aligned status area painted by `paint_status`
(`src/render/status.rs:28`). Four segments are drawn in order: battery, network, date, and time
(`status.rs:38`). The battery segment reads `mk_battery_status` and shows a percentage, or `AC` when no
battery is present or the reading is out of range (`src/state/indicators/battery.rs:19`). The network
segment shows `NET` when the DHCP client reports a bound lease and `OFF` otherwise
(`status.rs:33`, `src/state/indicators/net.rs:28`). The date is `YYYY-MM-DD` and the time is `HH:MM`,
both from the RTC; the time is 12-hour or 24-hour depending on the `policy` service's clock format field
(`src/state/indicators/clock.rs:19`, `src/state/indicators/clock.rs:40`,
`src/state/indicators/policy.rs:29`). The clock and indicators refresh once a second in the loop
(`src/server/runner/run.rs:38`, `src/server/runner/refresh_clock.rs:24`). There are no click targets in
the status area; the indicators are display-only. The menu-bar title reads `NONOS launcher`
(`src/render/chrome.rs:35`).

### Notifications, tray, and spotlight

Toasts, tray icons, and the spotlight panel are driven by other capsules over the shell's service port,
not by direct user clicks. A `NOTIFY` request enqueues a toast that appears above the dock for 4 seconds
(`src/server/handlers/notify.rs:25`, `src/state/toasts.rs:21`); the shell also raises its own toast
`network connected` the first time the link comes up (`refresh_clock.rs:29`). The tray is a set of
owner-scoped entries other capsules register, update, and remove (`TRAY_REGISTER`, `TRAY_UPDATE`,
`TRAY_REMOVE`). The spotlight is a panel toggled by the `SPOTLIGHT_OPEN` request
(`src/server/handlers/spotlight_open.rs:24`); it is drawn as a rectangle but its search UX is not yet
wired, so there is no in-panel interaction beyond the toggle.

## Architecture and lifecycle

The top-level modules are `compositor_client`, `input_router_client`, `market_client`, `protocol`,
`render`, `server`, `setup`, `state`, `wait_for_setup`, `wallpaper_client`, and `wm_client`
(`src/main.rs:22`).

The `Context` is the whole live state: the compositor, wm, input-router, and policy ports, the overlay
geometry and backing address, the tray table, the taskbar state, the spotlight state, the toast queue,
and a monotonically increasing request-id counter (`src/state/context.rs:19`,
`src/state/context.rs:43`). The taskbar tracks per-entry open and launch-pulse state, an active index,
and a visibility deadline (`src/state/taskbar/types.rs:20`). The tray table is a fixed array of 32
owner-tagged slots (`src/state/tray/entry.rs:19`, `src/state/tray/table.rs:20`). The toast queue holds
up to 3 live toasts, each truncated to 48 bytes with a 4-second lifetime (`src/state/toasts.rs:19`).

Lifecycle:

1. The kernel spawns the capsule at boot through the desktop fleet plan, which logs under the tag
   `DESKTOP-SHELL` and calls `spawn_desktop_shell_capsule`
   (`src/userspace/init/spawn_plan/desktop_fleet.rs:114`,
   `src/userspace/init/spawn_plan/boot.rs:20`). That path verifies the embedded ELF, id cert, manifest,
   and attestation, registers `desktop_shell` on port 4410 with the reply inbox on 4411, requests the
   five-capability mask, and marks the capsule alive
   (`src/userspace/capsule_desktop_shell/spawn.rs:40`, `spawn.rs:57`).
2. `wait_for_setup` retries `setup::run` until it succeeds (`src/wait_for_setup.rs:19`). One pass
   resolves and health-checks the peers, applies the wallpaper policy, allocates the overlay, builds the
   `Context`, paints the initial chrome, registers and commits the overlay surface, opens the taskbar
   popup window through the wm, and subscribes to wm lifecycle and input-router events
   (`src/setup/prime/run/run.rs:21`).
3. `paint_initial` retries up to eight times to paint the chrome and land the first full-screen damage
   commit (`src/server/paint_initial.rs:23`).
4. The loop drains inbound frames, refreshes the clock and indicators once a second, re-subscribes to
   input and wm if either subscription was lost, expires toasts and taskbar pulses, and blocks on the
   display vsync (`src/server/runner/run.rs:35`). It blocks the receive for up to 1000 ms when idle and
   polls at 16 ms while a subscription is still pending (`src/server/runner/constants.rs:18`,
   `src/server/runner/drain.rs:27`).

Inbound frames are classified in `drain`: a wm lifecycle notification and an input frame are recognised
by their own magics and handled first, and anything else is parsed as an `NDSH` request and dispatched
(`src/server/runner/drain.rs:39`, `drain.rs:42`, `drain.rs:45`). `dispatch` routes the six ops and
answers an unknown empty-body op with `E_BAD_OP` and a malformed one with `E_INVAL`
(`src/server/dispatch.rs:24`).

## Protocol and IPC

The shell serves six operations on its `NDSH` port (`src/protocol/ops.rs:17`):

```
  OP_HEALTHCHECK     0x0001   liveness probe, empty body      ops.rs:17
  OP_TRAY_REGISTER   0x0002   add an owner-scoped tray entry  ops.rs:18
  OP_TRAY_UPDATE     0x0003   relabel an owned tray entry     ops.rs:19
  OP_TRAY_REMOVE     0x0004   drop an owned tray entry        ops.rs:20
  OP_NOTIFY          0x0005   enqueue a toast                 ops.rs:21
  OP_SPOTLIGHT_OPEN  0x0006   toggle the spotlight panel      ops.rs:22
```

`TRAY_REGISTER` takes a `tray_id`, a label length, and up to 24 label bytes; it rejects a bad length with
`E_INVAL`, a duplicate `(pid, tray_id)` with `E_BUSY`, and a full table with `E_NOMEM`, then repaints the
menu bar and commits damage (`src/server/handlers/tray_register.rs:23`, `tray_register.rs:52`,
`tray_register.rs:56`). `TRAY_UPDATE` relabels an entry the caller owns or returns `E_NOENT`
(`src/server/handlers/tray_update.rs:40`). `TRAY_REMOVE` drops an owned entry or returns `E_NOENT`
(`src/server/handlers/tray_remove.rs:32`). `NOTIFY` validates a level and a body up to 128 bytes, pushes
a toast, and repaints (`src/server/handlers/notify.rs:38`, `notify.rs:48`). Every reply is a status word
written with `respond::status` (`src/server/respond.rs:21`). Error codes are the usual negatives:
`E_NOENT -2`, `E_NOMEM -12`, `E_BUSY -16`, `E_INVAL -22`, `E_BAD_OP -38`, plus header errors `E_BAD_MAGIC`,
`E_BAD_LEN`, `E_BAD_VERSION` from the parser (`src/protocol/errno.rs:17`, `src/protocol/decode.rs:19`).

Everything the shell reaches outward is an IPC call to another service. The calls it makes:

Compositor, service `compositor`, magic `NCMP` `0x4E43_4D50` (`src/compositor_client/wire.rs:10`):

```
  OP 0x0002   scene_submit    publish the overlay surface at z=1   scene_submit.rs:19
  OP 0x0003   damage_commit   present a changed rectangle          damage_commit.rs:19
  OP 0x0008   display_info    query width/height/stride/format     display_info.rs:21
  OP 0x0001   healthcheck     liveness probe                       health.rs
```

Window manager, service `wm`, magic `NWMP` `0x4E57_4D50`:

```
  OP 0x0002   window_open           open the taskbar popup window    wm_client/window_open.rs:26
  OP 0x0007   window_raise          raise the taskbar over an app    wm_client/window_raise.rs:26
  OP 0x0008   lifecycle_subscribe   subscribe to open/close events   wm_client/lifecycle_subscribe.rs:25
  OP 0x0001   healthcheck           liveness probe                   wm_client/mod.rs:36
```

The wm lifecycle notification is a separate inbound frame, magic `NWMV` `0x4E57_4D56`, carrying an event
kind (opened=0, closed=1), the owner pid, and a window id (`src/server/wm_notify.rs:26`,
`wm_notify.rs:37`). On an app window opening the shell raises its own taskbar window over it
(`wm_notify.rs:50`), resolves the owner pid back to a dock index and flips that entry's open state
(`wm_notify.rs:52`, `src/server/wm_notify_app_index.rs:21`), and raises a toast for the event
(`wm_notify.rs:56`).

Input router, service `input_router`, magic `NIRS` `0x4E49_5253`, `OP_SUBSCRIBE` = 2 with a kind mask
(`src/input_router_client.rs:26`, `input_router_client.rs:29`). The shell subscribes to key-down,
key-up, pointer-abs, wheel, button-down, button-up, and touch (`src/setup/prime/run/input_mask.rs:25`).
Inbound input frames arrive as `NINP` `0x4E49_4E50` and are decoded in `src/server/input.rs:29`.

Wallpaper, service `wallpaper`, magic `NWLP` `0x4E57_4C50`, `OP_SET_POLICY` = 4; setup pushes a policy
value so the wallpaper is in place before the chrome is submitted
(`src/wallpaper_client/mod.rs:26`, `src/setup/prime/run/apply_wallpaper_policy.rs:21`).

Market, service `market.index`, magic `NMKT` `0x4E4D_4B54`, `OP_HEALTHCHECK` = 6; the market is a
best-effort peer, so a zero port short-circuits to success and a missing market disables the probe
(`src/market_client/mod.rs:26`, `market_client/mod.rs:30`, `src/setup/discover/try_market.rs:19`).

Policy, service `policy`, `OP_GET` = 1, field `CLOCK_FORMAT24` = `0x0118`, used only to choose 12h vs 24h
in the clock (`src/state/indicators/policy.rs:23`, `policy.rs:24`). Network state is read from
`net.dhcp.client`, magic `NDHC` `0x4E44_4843`, `OP_LEASE_STATUS` = 3 (`src/state/indicators/net.rs:19`).

The overlay surface itself is registered and shared through the microkernel surface calls, not a service:
`mk_surface_register`, `mk_surface_share`, and `mk_surface_release` on failure
(`src/setup/prime/register.rs:41`, `register.rs:45`, `register.rs:59`).

## Security analysis

The shell looks like the most privileged capsule on the desktop because it coordinates everyone, but its
authority is exactly the app envelope. Its mask `0x1819` grants CoreExec, IPC, Memory,
GraphicsDisplayQuery, and GraphicsSurfaceCreate and nothing else
(`Capsule.mk:16`, `src/userspace/capsule_desktop_shell/spawn.rs:50`). It holds no Network, FileSystem,
hardware, driver, DMA, or `GraphicsPresent` capability.

- **The shell coordinates, it does not command.** It reaches the compositor, wm, input router, wallpaper,
  market, and policy through a service lookup and a normal request, so its authority over each is
  whatever that peer's own handler grants a caller. It cannot present a frame itself; like every other
  client it holds `GraphicsSurfaceCreate`, submits the overlay scene once
  (`src/setup/prime/register.rs:57`), and then commits damage for the compositor to present
  (`src/compositor_client/damage_commit.rs:22`). It cannot take an exclusive input grab; it only
  subscribes to the input router (`src/input_router_client.rs:29`).
- **The launcher focuses, it does not spawn.** A dock click resolves the target service and sends a
  single `NCTL` focus-self frame to its pid (`src/server/handlers/launcher_request.rs:26`). There is no
  installer call and no process creation in the shell; an app that is not already running cannot be
  brought up from the dock. Contrast the [terminal](terminal/README.md), which is the capsule that can ask
  the installer to spawn a store capsule.
- **Tray entries are owner-scoped on write.** `TRAY_REGISTER` tags each entry with `sender_pid` and
  rejects a duplicate `(pid, tray_id)` with `E_BUSY`; labels are length-checked against `TRAY_LABEL_MAX`
  and the table is a bounded 32 slots that returns `E_NOMEM` when full
  (`src/server/handlers/tray_register.rs:36`, `tray_register.rs:52`, `src/state/tray/entry.rs:19`).
  `TRAY_UPDATE` and `TRAY_REMOVE` only touch an entry the caller owns, returning `E_NOENT` otherwise
  (`src/server/handlers/tray_update.rs:40`, `src/server/handlers/tray_remove.rs:32`).
- **Honest boundary: the chrome is a shell-owned overlay.** The shell paints its chrome directly into the
  overlay's backing memory and trusts its own layout math to stay inside the overlay rectangle; the
  compositor clips the overlay as a whole scene but does not police what the shell writes within it
  (`src/render/chrome.rs:32`, `src/render/fill.rs`). A parsing or layout bug in the shell cannot escalate
  past a graphics client, but it can corrupt its own overlay pixels.

Isolation from other capsules is the kernel's, not the shell's: it is a CPL 3 user binary that only
speaks IPC and owns one surface, verified and enrolled at spawn like every other capsule
(`src/userspace/capsule_desktop_shell/spawn.rs:40`).

## How to contribute

The source lives at `userland/capsule_desktop_shell/`. The frame protocol is under `src/protocol/`, the
inbound handlers under `src/server/`, the renderer under `src/render/`, the live state under
`src/state/`, the setup sequence under `src/setup/`, and the outbound clients at the crate root
(`src/compositor_client/`, `src/wm_client/`, and so on).

To add an app to the launcher dock:

1. Add a `LauncherApp` entry to the `LAUNCHER_APPS` array with a `LauncherIcon`, a label, and the
   `app.*` service handle it should focus (`src/state/apps.rs:36`). Add the matching `LauncherIcon`
   variant (`apps.rs:18`) and its bitmap under `src/render/icons/`, wired into `draw_app_icon`
   (`src/render/icons.rs`).
2. The dock geometry, the taskbar open/pulse arrays, and the hit-test all size themselves from the array
   length (`src/render/layout.rs:21`, `src/state/taskbar/types.rs:17`), so no other constant needs to
   change. Confirm the new column still fits the dock width; `bottom_dock_rect` clamps the dock to the
   display (`src/render/layout.rs:38`).

To change the status indicators, edit the segment set and order in `paint_status`
(`src/render/status.rs:38`) and the per-indicator source under `src/state/indicators/` (`battery.rs`,
`net.rs`, `clock.rs`, `policy.rs`).

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_desktop_shell/Capsule.mk:19`):

```
  make nonos-mk-desktop-shell              build the capsule ELF
  make nonos-mk-desktop-shell-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-desktop-shell-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-desktop-shell-keys   check the per-capsule signing keys exist
```

For a running desktop that includes the shell, `make nonos-mk-desktop-gui-prod` and
`make nonos-mk-full-gui-prod` build the desktop and full GUI kernel images; both bundle the
`desktop-shell` artifacts (`Makefile:1067`, `Makefile:1093`, `Makefile:1079`, `Makefile:1109`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns errors as a status word, and the workspace release
profile is `panic = "abort"`, `Cargo.toml:826`); modular files, one unit per file, with `mod.rs` used
only for re-exports (as in `src/server/handlers/mod.rs` and `src/state/mod.rs`); and the AGPL header at
the top of every source file, matching the header on every existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[DESKTOP-SHELL] capsule spawned` from the boot log (tag `DESKTOP-SHELL`, message `capsule spawned`,
`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line means the
capsule never started, usually a signature, manifest, capability, or attestation failure; the error path
prints an `[ERROR]` line built from the spawn error instead
(`src/userspace/init/capsule_boot/run.rs:32`, `src/userspace/init/capsule_boot/error.rs:21`).

Failure modes and where to look:

- The shell never appears and setup never completes. `wait_for_setup` loops on `setup::run` and only
  returns once every required peer resolves and health-checks (`src/wait_for_setup.rs:19`,
  `src/setup/prime/peers.rs:28`). The wallpaper lookup is required and fails setup with
  `wallpaper service not announced` if the wallpaper capsule is not up
  (`src/setup/discover/require_wallpaper.rs:19`, `src/setup/prime/run/apply_wallpaper_policy.rs:19`), and
  the compositor, input router, and wm are all required (`peers.rs:29`). So bring-up ordering is that the
  compositor, wm, input router, and wallpaper register before the shell. The market is best-effort and
  its absence never blocks setup (`src/setup/discover/try_market.rs:19`).
- The shell is up but the dock will not draw. The dock only paints when it is visible, and it auto-hides;
  reveal it by moving the pointer into the bottom band or clicking near the bottom of the screen
  (`src/render/chrome.rs:36`, `src/server/input.rs:70`). If it never reveals, the input subscription is
  the suspect: the loop re-subscribes each second while `input_ready` is false
  (`src/server/runner/run.rs:40`, `src/setup/prime/run/subscribe_input.rs:19`).
- An app will not launch from the dock. The dock focuses, it does not spawn, so the target must already
  be running. `launcher_request` needs a live service registration; a missing pid makes the click a
  no-op (`src/server/handlers/launcher_request.rs:36`). The dock's open/active state also depends on the
  wm lifecycle subscription, re-armed each second while `wm_notify_ready` is false
  (`src/server/runner/run.rs:44`, `src/setup/prime/run/subscribe_wm.rs:20`).
- The splash never leaves the screen. The [boot splash](boot-splash.md) polls `lookup("desktop_shell")`
  and hands off once it resolves (`userland/capsule_boot_splash/src/main.rs:103`), so a splash that never
  clears usually means the shell never registered its service; check for the `[DESKTOP-SHELL]` boot line
  above.
- Tray, notify, or spotlight errors on the wire. A bad body length or an empty or oversized label is
  `E_INVAL`, a duplicate tray id for the same owner is `E_BUSY`, a full tray table is `E_NOMEM`, and an
  update or remove of an id the caller does not own is `E_NOENT`
  (`src/server/handlers/tray_register.rs:24`, `tray_register.rs:53`, `tray_register.rs:57`,
  `src/server/handlers/tray_update.rs:41`). A market health-check failure surfaces as `market call
  failed` inside the shell rather than a protocol error to a tray caller, because the market is a
  best-effort peer (`src/market_client/mod.rs:51`).

## Source map

```
  src/main.rs                                   _start -> heap, wait_for_setup, server::run
  src/wait_for_setup.rs                         retry setup::run until every peer is up
  src/setup/prime/run/run.rs                    the setup sequence (peers, overlay, windows, subscribe)
  src/setup/prime/register.rs                   register + submit the overlay scene at z=1
  src/setup/discover/                           peer lookup, require_wallpaper, try_market
  src/server/runner/run.rs                      the loop (drain, clock, expiry, vsync)
  src/server/runner/drain.rs                    frame classification (wm-notify, input, NDSH request)
  src/server/dispatch.rs                        NDSH op routing
  src/server/handlers/                          health, tray_register/update/remove, notify, spotlight_open
  src/server/handlers/launcher_focus.rs         dock hit-test
  src/server/handlers/launcher_request.rs       NCTL focus-self to the target app
  src/server/input.rs                           NINP decode, dock reveal/collapse, launch click
  src/server/wm_notify.rs                        NWMV lifecycle: open/close -> dock state + toast
  src/state/apps.rs                             LAUNCHER_APPS: the nine dock entries
  src/state/context.rs                          ports, overlay geometry, tray, taskbar, toasts
  src/state/taskbar/                             open/pulse/active/visible state and expiry
  src/state/tray/                                the 32-slot owner-scoped tray table
  src/state/toasts.rs                            the 3-slot toast queue
  src/state/indicators/                          battery, net, clock, policy status sources
  src/render/chrome.rs                           the composite paint (menu bar, dock, badges, spotlight)
  src/render/bottom_taskbar.rs                   the dock entries and their tints
  src/render/status.rs                           the right-aligned status segments
  src/compositor_client/                         scene_submit, damage_commit, display_info, health
  src/wm_client/                                 window_open, window_raise, lifecycle_subscribe, health
  src/input_router_client.rs                     input subscribe
  src/wallpaper_client/, src/market_client/      the best-effort peers
  src/protocol/                                  NDSH header, ops, limits, parse, respond, errno
  Capsule.mk                                     slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_desktop_shell/spawn.rs  the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/desktop_fleet.rs  the desktop-fleet spawn entry
  nonos-mk/capsule.mk                            the generated nonos-mk-desktop-shell[-sign|-verify] targets
```

Every reference above is verified against those trees.

# Application Capsules

This page documents the user-facing application capsules and the app skeleton
contract they use. Read [Capsule Inventory](capsules.md), [Desktop](desktop.md),
and [SDK](sdk.md) first.

Audit an app from the runner inward: entry, manifest, window request, input
subscription, repaint, and teardown. That path keeps the desktop behavior tied
to code instead of screenshots or intent.

---

## 1. App skeleton contract

Eight application capsules use `nonos_app_skeleton::run`: about, calculator,
terminal, file manager, text editor, settings, process manager, and input proof
(`userland/capsule_about/src/main.rs:24`, `userland/capsule_calculator/src/main.rs:24`,
`userland/capsule_terminal/src/main.rs:27`, `userland/capsule_file_manager/src/main.rs:24`,
`userland/capsule_text_editor/src/main.rs:24`, `userland/capsule_settings/src/main.rs:24`,
`userland/capsule_process_manager/src/main.rs:24`, `userland/capsule_input_proof/src/main.rs:24`).
The skeleton `App` trait requires a manifest, an input event handler, and a
paint function (`userland/app_skeleton/src/app/behavior.rs:21`). The manifest
stores title, window id, window kind, initial x and y, width, height, and input
kind mask (`userland/app_skeleton/src/app/manifest.rs:20`).

```
+--------------------------+
| capsule entry            |
| run App constructor      |
+------------+-------------+
             |
+------------+-------------+
| app skeleton             |
| heap peers manifest      |
+------------+-------------+
             |
+------------+-------------+
| window open request      |
| initial scene submit     |
+------------+-------------+
             |
+------------+-------------+
| input subscription       |
| service frame loop       |
+------------+-------------+
             |
+------------+-------------+
| app event handler        |
| repaint close teardown   |
+--------------------------+
```

The runner initializes heap, resolves peers, constructs the app, reads the
manifest, boots the app, allocates an IPC buffer, and then repeatedly calls the
service frame (`userland/app_skeleton/src/runner/entry.rs:30`). Boot opens the
window, subscribes to input, and primes the first frame
(`userland/app_skeleton/src/runner/boot.rs:33`). Opening a window allocates
backing storage, registers and shares a surface, announces the window to the
WM, and returns a binding with the placement accepted by the WM
(`userland/app_skeleton/src/setup/open.rs:26`). Announce sends the manifest
window id, kind, initial position, and requested size to `wm::window_open`,
then submits the scene and subscribes to input
(`userland/app_skeleton/src/setup/announce.rs:27`).

The service frame refreshes the input subscription, ensures the first paint has
reached the compositor, drains input, closes the app if the result requests it,
repaints when needed, and waits for display vsync
(`userland/app_skeleton/src/runner/service_frame.rs:28`). Decoration close is
implemented by hit-testing button-down input against the toolkit close button
(`userland/app_skeleton/src/runner/decorations.rs:28`). Closing removes the
scene, clears the input subscription, releases the surface, unmaps backing
memory, closes the WM window, and exits (`userland/app_skeleton/src/runner/teardown.rs:25`).

## 2. Manifest placement and input masks

These values are not layout suggestions. They are compiled app manifest values.
The WM can still return an adjusted placement, but the request sent by the app
comes from the table below.

| Capsule | Title | Window id | Initial position | Size | Input mask | Source |
|---------|-------|-----------|------------------|------|------------|--------|
| `app.about` | `About NONOS` | `0x4142_4F55` | `WINDOW_INITIAL_X`, `WINDOW_INITIAL_Y` from theme | `WINDOW_WIDTH` by `WINDOW_HEIGHT` | key down, button down, pointer abs | `userland/capsule_about/src/about/manifest.rs:21`, `userland/capsule_about/src/about/manifest.rs:25`, `userland/capsule_about/src/about/manifest.rs:26`, `userland/capsule_about/src/about/manifest.rs:28`; theme values at `userland/capsule_about/src/about/theme.rs:27` to `userland/capsule_about/src/about/theme.rs:30` |
| `app.calculator` | `Calculator` | `0x4341_4C43` | `876`, `92` | `360` by `520` | key down, button down, pointer abs | `userland/capsule_calculator/src/calc/manifest.rs:19`, `userland/capsule_calculator/src/calc/manifest.rs:21`, `userland/capsule_calculator/src/calc/manifest.rs:22`, `userland/capsule_calculator/src/calc/manifest.rs:28` |
| `app.terminal` | `Terminal` | `0x5445_524D` | `188`, `404` | `520` by `300` | key down | `userland/capsule_terminal/src/term/manifest.rs:19`, `userland/capsule_terminal/src/term/manifest.rs:22`, `userland/capsule_terminal/src/term/manifest.rs:24` |
| `app.file_manager` | `File Manager` | `0x464D_4752` | `792`, `438` | `360` by `260` | key down, button down, pointer abs | `userland/capsule_file_manager/src/fm/manifest.rs:19`, `userland/capsule_file_manager/src/fm/manifest.rs:22`, `userland/capsule_file_manager/src/fm/manifest.rs:26` |
| `app.text_editor` | `Text Editor` | `0x5445_4458` | `346`, `220` | `500` by `320` | key down | `userland/capsule_text_editor/src/editor/manifest.rs:19`, `userland/capsule_text_editor/src/editor/manifest.rs:22`, `userland/capsule_text_editor/src/editor/manifest.rs:24` |
| `app.settings` | `NONOS Settings` | `0x5345_5447` | `420`, `44` | `760` by `520` | key down | `userland/capsule_settings/src/settings/manifest.rs:19`, `userland/capsule_settings/src/settings/manifest.rs:22`, `userland/capsule_settings/src/settings/manifest.rs:24` |
| `app.process_manager` | `Process Manager` | `0x504D_4752` | `744`, `456` | `440` by `240` | key down | `userland/capsule_process_manager/src/pm/manifest.rs:19`, `userland/capsule_process_manager/src/pm/manifest.rs:22`, `userland/capsule_process_manager/src/pm/manifest.rs:24` |
| `app.input_proof` | `Input Proof` | `0x494E_5052` | `0`, `0` | `1024` by `768` | key down, pointer rel, pointer abs, button down | `userland/capsule_input_proof/src/proof/manifest.rs:19`, `userland/capsule_input_proof/src/proof/manifest.rs:21`, `userland/capsule_input_proof/src/proof/manifest.rs:23`, `userland/capsule_input_proof/src/proof/manifest.rs:29` |

## 3. Application behavior surface

| Capsule | Runtime contract | Behavior refs |
|---------|------------------|---------------|
| `app.about` | Implements the app skeleton trait, routes key and pointer input, paints header, tabs, body, scrollbar, and status bar. | `userland/capsule_about/src/about/app.rs:34`, `userland/capsule_about/src/about/event/router.rs:34`, `userland/capsule_about/src/about/paint/frame.rs:27` |
| `app.calculator` | Implements the app skeleton trait, routes key and pointer button input, paints calculator background, display, grid, memory badge, and wordmark. | `userland/capsule_calculator/src/calc/app.rs:34`, `userland/capsule_calculator/src/calc/event/router.rs:23`, `userland/capsule_calculator/src/calc/paint/frame.rs:26` |
| `app.terminal` | Implements the app skeleton trait, handles key events, command entry, clipboard paste, line copy, and terminal painting. | `userland/capsule_terminal/src/term/terminal/app_impl.rs:21`, `userland/capsule_terminal/src/event/on_event.rs:22`, `userland/capsule_terminal/src/event/on_enter.rs:25`, `userland/capsule_terminal/src/paint/paint.rs:27` |
| `app.file_manager` | Implements the app skeleton trait, handles keyboard and pointer selection, and paints file manager state. | `userland/capsule_file_manager/src/fm/app.rs:37`, `userland/capsule_file_manager/src/fm/event.rs:22`, `userland/capsule_file_manager/src/fm/paint.rs:26` |
| `app.text_editor` | Implements the app skeleton trait, handles text input, close, ctrl actions, clipboard copy, clipboard paste, VFS open, and VFS save. | `userland/capsule_text_editor/src/editor/app.rs:34`, `userland/capsule_text_editor/src/editor/event.rs:22`, `userland/capsule_text_editor/src/editor/on_ctrl.rs:25`, `userland/capsule_text_editor/src/editor/ctrl_open.rs:22`, `userland/capsule_text_editor/src/editor/ctrl_save.rs:22` |
| `app.settings` | Implements the app skeleton trait, switches between browsing and editing event paths, and paints policy categories, values, status, and tabs. | `userland/capsule_settings/src/settings/app.rs:56`, `userland/capsule_settings/src/settings/event/on_event.rs:25`, `userland/capsule_settings/src/settings/event/on_event_browsing.rs:30`, `userland/capsule_settings/src/settings/event/on_event_editing.rs:24`, `userland/capsule_settings/src/settings/paint/paint.rs:31` |
| `app.process_manager` | Implements the app skeleton trait, handles button-down, escape, and repaint-driving key events, and paints process manager state. | `userland/capsule_process_manager/src/pm/app.rs:34`, `userland/capsule_process_manager/src/pm/event.rs:21`, `userland/capsule_process_manager/src/pm/paint.rs:25` |
| `app.input_proof` | Implements the app skeleton trait, records key, pointer, and click markers, and paints the proof view. | `userland/capsule_input_proof/src/proof/app.rs:36`, `userland/capsule_input_proof/src/proof/markers.rs:24`, `userland/capsule_input_proof/src/proof/app.rs:46` |

## 4. Interaction matrix

The matrix is behavior, not product language. It records what happens after an
input frame reaches a skeleton app, who gets focus, where close is decided, and
which path can repaint.

```
+--------------------------+
| IPC delivery             |
| control frames first     |
+------------+-------------+
             |
+------------+-------------+
| parse Delivery           |
| normalize decorations    |
+------------+-------------+
             |
+------------+-------------+
| click focus request      |
| decoration close test    |
+------------+-------------+
             |
+------------+-------------+
| App on event             |
| repaint or close         |
+------------+-------------+
             |
+------------+-------------+
| scene remove             |
| window close             |
+--------------------------+
```

Every skeleton app gets the same outer close and focus behavior before its own
handler runs. The runner normalizes touch into button-down, raises focus on
button input, treats a toolkit close-button hit as close, and also closes when
the app handler returns `EventOutcome::Close`
(`userland/app_skeleton/src/runner/decorations.rs:21`,
`userland/app_skeleton/src/runner/decorations.rs:28`,
`userland/app_skeleton/src/runner/drain_ipc.rs:51`,
`userland/app_skeleton/src/runner/drain_ipc.rs:52`,
`userland/app_skeleton/src/runner/drain_ipc.rs:53`,
`userland/app_skeleton/src/runner/drain_ipc.rs:56`). Closing removes the
compositor scene, clears the input subscription, releases the surface, unmaps the
backing memory, closes the WM window, and exits
(`userland/app_skeleton/src/runner/teardown.rs:31`,
`userland/app_skeleton/src/runner/teardown.rs:35`,
`userland/app_skeleton/src/runner/teardown.rs:36`,
`userland/app_skeleton/src/runner/teardown.rs:39`,
`userland/app_skeleton/src/runner/teardown.rs:43`,
`userland/app_skeleton/src/runner/teardown.rs:46`).

| Capsule | Accepted input | Close path | Result contract |
|---------|----------------|------------|-----------------|
| `app.about` | Button-down routes to the pointer tab handler. Key-down handles escape, tab, shift-tab, up, down, page-up, page-down, home, and end. Non key-down events are idle. | Escape returns close through `on_esc`; toolkit close decoration is handled by the skeleton. | Navigation keys return repaint through the page, tab, home, end, and arrow handlers. Source: `userland/capsule_about/src/about/event/router.rs:34` to `userland/capsule_about/src/about/event/router.rs:51`, close source at `userland/capsule_about/src/about/event/on_esc.rs:19`. |
| `app.calculator` | Button-down hit-tests the calculator grid. Key-down is classified into digits, decimal, operators, equals, clear, negate, percent, square root, square, reciprocal, and memory actions. Non key-down events are idle. | Escape is classified as close. | Valid key or pointer actions run the calculator dispatcher and return repaint. Source: `userland/capsule_calculator/src/calc/event/router.rs:23` to `userland/capsule_calculator/src/calc/event/router.rs:30`, `userland/capsule_calculator/src/calc/event/key_classifier.rs:26` to `userland/capsule_calculator/src/calc/event/key_classifier.rs:53`, `userland/capsule_calculator/src/calc/event/on_pointer_button.rs:24` to `userland/capsule_calculator/src/calc/event/on_pointer_button.rs:37`. |
| `app.terminal` | Only key-down events reach terminal logic. The key handler covers escape, enter, backspace, delete, arrows, home, end, page-up, page-down, printable ASCII, and ctrl-modified keys. | Escape returns close. | Ctrl-V pastes, Ctrl-Shift-C copies the current line, Ctrl-L clears scrollback, Ctrl-C clears the line and emits `^C`, Ctrl-U clears the line, Ctrl-A moves home, and Ctrl-E moves end. Source: `userland/capsule_terminal/src/event/on_event.rs:22` to `userland/capsule_terminal/src/event/on_event.rs:27`, `userland/capsule_terminal/src/event/on_key.rs:31` to `userland/capsule_terminal/src/event/on_key.rs:64`, `userland/capsule_terminal/src/event/on_ctrl.rs:36` to `userland/capsule_terminal/src/event/on_ctrl.rs:67`. |
| `app.file_manager` | Button-down maps the y coordinate into a row and then falls through as enter. Key-down handles escape, enter, backspace, left, up, down, right, and vim-style `j`, `k`, `h`, `l`. | Escape returns close. | Enter descends into directories or records a selected file path as preview state, parent navigation truncates the prefix, cursor movement repaints. Source: `userland/capsule_file_manager/src/fm/event.rs:22` to `userland/capsule_file_manager/src/fm/event.rs:70`. |
| `app.text_editor` | Only key-down events are accepted. Ctrl keys route to copy, open, save, and paste. Plain keys handle escape, backspace, enter, and Unicode scalar insertion. | Escape returns close. | Mutating text sets the status to `edited /notes.txt` and returns repaint. Source: `userland/capsule_text_editor/src/editor/event.rs:22` to `userland/capsule_text_editor/src/editor/event.rs:48`, ctrl routing at `userland/capsule_text_editor/src/editor/on_ctrl.rs:25` to `userland/capsule_text_editor/src/editor/on_ctrl.rs:33`. |
| `app.settings` | Button-down routes through the pointer handler. Browsing mode handles escape, tab, arrows, space, enter, `[`, and `]`. Editing mode handles escape, enter, backspace, and text input. | Browsing escape returns close. Editing escape cancels editing and repaints. | Browsing changes category, cursor, value, or edit state. Editing commits or cancels string edits. Source: `userland/capsule_settings/src/settings/event/on_event.rs:25` to `userland/capsule_settings/src/settings/event/on_event.rs:35`, `userland/capsule_settings/src/settings/event/on_event_browsing.rs:30` to `userland/capsule_settings/src/settings/event/on_event_browsing.rs:42`, `userland/capsule_settings/src/settings/event/on_event_editing.rs:24` to `userland/capsule_settings/src/settings/event/on_event_editing.rs:40`. |
| `app.process_manager` | Button-down refreshes. Any key-down except escape refreshes. Other events are idle. | Escape returns close. | Refresh returns repaint and the paint path renders current process state. Source: `userland/capsule_process_manager/src/pm/event.rs:21` to `userland/capsule_process_manager/src/pm/event.rs:33`, paint source at `userland/capsule_process_manager/src/pm/paint.rs:25`. |
| `app.input_proof` | Key-down, pointer-relative, pointer-absolute, and button-down events are latched. | No app-specific close path is implemented; toolkit close decoration is still handled by the skeleton. | First paint emits `surface composited`; first delivery emits `surface ready`; later input can emit key, motion, click, focus routed, and pass markers. Source: `userland/capsule_input_proof/src/proof/app.rs:41` to `userland/capsule_input_proof/src/proof/app.rs:52`, marker source at `userland/capsule_input_proof/src/proof/markers.rs:22` to `userland/capsule_input_proof/src/proof/markers.rs:69`. |

## 5. Direct GUI apps

`app.input_probe` and `app.setup_wizard` do not use `nonos_app_skeleton`.
Both initialize heap, run a custom setup function, and enter
`server::runner::run` (`userland/capsule_input_probe/src/main.rs:16`,
`userland/capsule_setup_wizard/src/main.rs:16`). Their setup paths resolve the
compositor and input router, query display info, allocate and register an
ARGB8888 surface, share it, submit a full-screen overlay scene, and commit
damage (`userland/capsule_input_probe/src/setup/mod.rs:18`,
`userland/capsule_setup_wizard/src/setup/mod.rs:18`). The input probe runner
subscribes to input, grabs keyboard, receives input deliveries, and draws only
printable key-down events (`userland/capsule_input_probe/src/server/runner.rs:12`).
The setup wizard runner subscribes, grabs keyboard, redraws wizard screens,
advances on key events, removes its compositor scene on completion, and exits
(`userland/capsule_setup_wizard/src/server/runner.rs:12`). The wizard step
logic defines `DONE = 10`, enter as advance, escape as back, and list
navigation over `k`, `j`, and number keys
(`userland/capsule_setup_wizard/src/server/step.rs:1`).

`app.input_probe` only draws printable key-down events after subscribing to the
input router and grabbing the keyboard
(`userland/capsule_input_probe/src/server/runner.rs:12`,
`userland/capsule_input_probe/src/server/runner.rs:13`,
`userland/capsule_input_probe/src/server/runner.rs:14`,
`userland/capsule_input_probe/src/server/runner.rs:31`,
`userland/capsule_input_probe/src/server/runner.rs:37`). `app.setup_wizard`
also subscribes and grabs keyboard, but it redraws a screen after each accepted
key, applies a step transition, removes its scene at completion, and exits
(`userland/capsule_setup_wizard/src/server/runner.rs:12`,
`userland/capsule_setup_wizard/src/server/runner.rs:13`,
`userland/capsule_setup_wizard/src/server/runner.rs:14`,
`userland/capsule_setup_wizard/src/server/runner.rs:29`,
`userland/capsule_setup_wizard/src/server/runner.rs:30`,
`userland/capsule_setup_wizard/src/server/runner.rs:31`,
`userland/capsule_setup_wizard/src/server/runner.rs:32`,
`userland/capsule_setup_wizard/src/server/runner.rs:33`,
`userland/capsule_setup_wizard/src/server/runner.rs:35`). The shared step helper
caps the done state at `10`, treats enter as advance, escape as back, and maps
`k`, `j`, and number keys into list movement
(`userland/capsule_setup_wizard/src/server/step.rs:1`,
`userland/capsule_setup_wizard/src/server/step.rs:14`,
`userland/capsule_setup_wizard/src/server/step.rs:22`,
`userland/capsule_setup_wizard/src/server/step.rs:30`).

```
+--------------------------+
| direct app main          |
| heap and GUI client      |
+------------+-------------+
             |
+------------+-------------+
| display info             |
| surface create           |
+------------+-------------+
             |
+------------+-------------+
| scene submit             |
| input subscribe          |
+------------+-------------+
             |
+------------+-------------+
| server runner            |
| key pointer handler      |
+------------+-------------+
             |
+------------+-------------+
| scene remove             |
| app exit                 |
+--------------------------+
```

These two apps are useful direct GUI exercises because they drive the raw GUI
client, compositor, and input route without `nonos_app_skeleton` in the middle.

## 6. App boot inclusion

The current init entry path spawns the app fleet directly. After network,
desktop, and market, `run_init` calls `spawn_apps` before it drops its own
priority and enters the residual loop (`src/userspace/init/entry.rs:31`,
`src/userspace/init/entry.rs:32`, `src/userspace/init/entry.rs:33`). The spawn
plan module exports `spawn_apps` from `app_orchestrator`, which walks the
`apps.rs` and `apps_tools.rs` fleets (`src/userspace/init/spawn_plan/mod.rs:38`,
`src/userspace/init/spawn_plan/apps.rs:17`). `spawn` in `apps.rs` brings up the
input proof capsule, about, hello, calculator, snake, wallet, terminal, and
file manager, then hands off to `apps_tools::spawn` for the process manager and
the rest of the tool fleet (`src/userspace/init/spawn_plan/apps.rs:17`,
`src/userspace/init/spawn_plan/apps_tools.rs:17`). Every one of these is a
feature-gated call into `boot::capsule`, so a disabled feature simply drops the
capsule from the fleet rather than failing the boot.

Because the apps are already running, there is no launch broker. The desktop
shell taskbar does not spawn anything on click. It looks up the clicked app's
service pid and, if the app is alive, sends that pid an `NCTL` focus control
frame; if the lookup fails it does nothing
(`userland/capsule_desktop_shell/src/server/handlers/launcher_request.rs:26`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_focus.rs:24`).

The setup wizard is spawned instead of the full desktop when
`microkernel-setup-wizard` is enabled without input probe, and the full desktop
plus market is spawned after the wizard exits
(`src/userspace/init/spawn_plan/orchestrator.rs:51`,
`src/userspace/init/spawn_plan/orchestrator.rs:63`). Input probe mode starts
only input router, compositor, and input probe in that desktop phase
(`src/userspace/init/spawn_plan/input_probe_fleet.rs:17`).

## 7. Security analysis

An application capsule is the least-trusted thing in the running system, and its
capability mask reflects that. The pure display-only apps (about, calculator,
hello, and the other skeleton apps) carry `0x1819`, which is `CoreExec`, `IPC`,
`Memory`, `GraphicsDisplayQuery`, and `GraphicsSurfaceCreate`
(`userland/capsule_about/Capsule.mk:12` and the matching rows in the inventory).
Apps that move real data widen the mask exactly where their job requires it and
nowhere else: `file_manager` and `text_editor` add `FileSystem` (`0x1859`),
`terminal` adds `Network`, `Crypto`, and `FileSystem` (`0x187d`), `wallet` and
`browser` add `Network` and `Crypto` (`0x183d`), `process_manager` adds
`ProcessControl` (`0x2001819`), and `settings` adds `Network` and `DeviceEnum`
(`0x981d`). For the display-only apps, `0x1819`
is exactly enough to run, talk to the desktop services over IPC, query the
display, register its own surface, and nothing more. An app holds no `Driver`,
`Mmio`, `Irq`, `Dma`, `InputSource`, or `Admin` bit, so every device, interrupt, and
admin syscall in the ABI returns `EPERM` for it at the gate
(`src/capabilities/types/defs.rs:18`). An app that tried to post its own input events or
reboot the machine would be refused by the kernel, not by the desktop. The input proof
capsule carries one extra bit, `0x1919` adds `Debug`, because it is a proof harness that
emits debug markers; the ordinary apps do not.

An app never grabs input for itself. Exclusive keyboard and pointer grabs are
reserved to three named system capsules, the boot splash, the setup wizard, and the
input probe, and the input router refuses a grab from anyone else with `E_ACCES`
(the gate is on the [input router page](input-router/README.md)). So a
skeleton app receives only the events the router routes to it by focus or hit test,
and cannot monopolize the keyboard. Focus itself is the WM's decision, not the app's:
the runner raises a focus request on button input, but the WM owns the focus state,
so an app cannot steal focus by asserting it.

The two direct GUI apps, the input probe and the setup wizard, are the exception that
proves the rule. They do grab the keyboard, and they can only do so because they are
on the router's trusted-grabber list by name. They still spawn through the same
verified path as every other capsule, so being trusted to grab input does not make
them trusted to skip signature and manifest verification.

## 8. Debugging an app

An app that launched but shows no window failed somewhere in boot, which opens the
window, subscribes to input, and primes the first frame
(`userland/app_skeleton/src/runner/boot.rs:33`). The order there is the debugging
map: open allocates backing, registers and shares a surface, and announces to the
WM (`userland/app_skeleton/src/setup/open.rs:26`); a window that never appears
usually failed at the announce or the first scene submit rather than in the app's own
paint. An app whose window appears but never repaints is almost always the first-paint
handshake or a missing damage commit; the service frame ensures the first paint
reached the compositor before it starts draining input
(`userland/app_skeleton/src/runner/service_frame.rs:28`).

An app that does not respond to keys is a subscription or focus question, not usually
an app-logic bug. Each app requests a specific input kind mask in its manifest (the
table in section 2), and the router delivers only kinds the app subscribed to, only
when the WM reports it focused. The terminal, editor, settings, and process manager
subscribe to key-down only, so pointer events reaching their handlers would be a
router or WM problem. When an app should close but does not, the close path is shared:
the runner treats a toolkit close-button hit and an `EventOutcome::Close` from the
handler the same way, and teardown removes the scene, clears the subscription,
releases the surface, unmaps backing, closes the WM window, and exits
(`userland/app_skeleton/src/runner/teardown.rs:25`). A window that closes visually but
leaves a stale service is a teardown that did not reach the exit.

The clicked-but-nothing-happens case is the taskbar, not the app. On this branch the
apps are already spawned at boot by `spawn_apps` (`src/userspace/init/spawn_plan/apps.rs:17`),
so a taskbar click does not launch anything; it looks up the app's service pid and, if
it resolves, sends that pid an `NCTL` focus frame
(`userland/capsule_desktop_shell/src/server/handlers/launcher_request.rs:26`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_focus.rs:24`). A click that
does nothing usually means the service lookup returned no pid, which means the app was
never spawned or has exited, not that focus failed.

## 9. Source map

```
  userland/app_skeleton/src/app/{behavior,manifest}.rs   the App trait and the manifest
  userland/app_skeleton/src/runner/{entry,boot,service_frame,teardown}.rs  the run loop
  userland/app_skeleton/src/setup/{open,announce}.rs     window open and WM announce
  userland/app_skeleton/src/runner/{decorations,drain_ipc}.rs  close and focus handling
  userland/capsule_<app>/src/<app>/{app,event,paint}.rs  each app's trait impl and handlers
  userland/capsule_input_probe/, capsule_setup_wizard/   the two direct GUI apps
  src/userspace/init/spawn_plan/apps.rs                   spawn_apps: the boot app fleet
  userland/capsule_desktop_shell/src/server/handlers/launcher_{request,focus}.rs  taskbar focus
```

The per-app capability masks and endpoints are in
[the capsule inventory](capsules.md); the input delivery contract is on
[the input router page](input-router/README.md).

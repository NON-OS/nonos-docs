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

These two apps are useful smoke tests because they exercise the raw GUI client,
compositor, and input route without `nonos_app_skeleton` in the middle.

## 6. App boot inclusion

The current init entry path registers the launcher broker, then calls desktop
and market after network
(`src/userspace/init/entry.rs:33`, `src/userspace/init/entry.rs:34`,
`src/userspace/init/entry.rs:35`, `src/userspace/init/entry.rs:36`). The
spawn plan module imports desktop, desktop services, input probe, drivers,
network, core, and smoketests, but does not import the app fleet modules or
export an app spawn entry (`src/userspace/init/spawn_plan/mod.rs:17`,
`src/userspace/init/spawn_plan/mod.rs:41`). `apps.rs` and `apps_tools.rs`
still define a source-level app fleet for input proof, about, calculator,
terminal, file manager, text editor, settings, and process manager
(`src/userspace/init/spawn_plan/apps.rs:17`,
`src/userspace/init/spawn_plan/apps_tools.rs:17`). In the current tree those
functions are not reached by `run_init`. Interactive launch goes through the
desktop shell launcher and the init-owned `desktop.launcher` broker: shell
focuses an existing app service when present, otherwise it sends a launch id to
the broker, which authorizes the request against the current desktop shell pid
and calls the allowlisted verified spawner
(`userland/capsule_desktop_shell/src/server/handlers/launcher_request/request.rs:19`,
`src/userspace/init/launcher/authorize.rs:19`,
`src/userspace/init/launcher/spawn.rs:17`).

The setup wizard is spawned instead of the full desktop when
`microkernel-setup-wizard` is enabled without input probe, and the full desktop
plus market is spawned after the wizard exits
(`src/userspace/init/spawn_plan/orchestrator.rs:51`,
`src/userspace/init/spawn_plan/orchestrator.rs:63`). Input probe mode starts
only input router, compositor, and input probe in that desktop phase
(`src/userspace/init/spawn_plan/input_probe_fleet.rs:17`).

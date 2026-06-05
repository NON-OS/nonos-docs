# Application Capsules

This page documents the user-facing application capsules and the app skeleton
contract they use. Read [Capsule Inventory](capsules.md), [Desktop](desktop.md),
and [SDK](sdk.md) first.

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
  +-----------------------+
  | capsule main          |
  | run(App::new)         |
  +-----------+-----------+
              |
  +-----------+-----------+
  | skeleton heap setup   |
  | peer discovery        |
  +-----------+-----------+
              |
  +-----------+-----------+
  | WM window_open        |
  | compositor scene      |
  | input subscribe       |
  +-----------+-----------+
              |
  +-----------+-----------+
  | service frame loop    |
  | input paint close     |
  +-----------------------+
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

## 2. Boot placement and input masks

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

## 4. Direct GUI apps

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

## 5. App boot inclusion

The standard app fleet starts input proof, about, calculator, terminal, and
file manager, then calls the tool app group for text editor, settings, and
process manager (`src/userspace/init/spawn_plan/apps.rs:17`,
`src/userspace/init/spawn_plan/apps_tools.rs:17`). The setup wizard is spawned
instead of the full desktop when `microkernel-setup-wizard` is enabled without
input probe, and the full desktop plus market is spawned after the wizard exits
(`src/userspace/init/spawn_plan/orchestrator.rs:51`,
`src/userspace/init/spawn_plan/orchestrator.rs:63`). Input probe mode starts
only input router, compositor, and input probe in that desktop phase
(`src/userspace/init/spawn_plan/input_probe_fleet.rs:17`).

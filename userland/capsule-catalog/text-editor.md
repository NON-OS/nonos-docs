# capsule_text_editor (full reference)

`capsule_text_editor` is the text editor in the NONOS desktop fleet: a GUI window over a single
fixed-capacity edit buffer that loads and saves a file through the vfs service, copies and pastes
through the clipboard service, and notifies the desktop shell when it saves. It is a focused single
document editor, not an IDE. There is no file explorer, no tab bar, no find or replace, no undo or
redo, and no selection; editing is append-and-backspace at the end of the buffer, and the arrow keys
scroll the view rather than move an insertion caret. This is the exhaustive reference; the source is
`userland/capsule_text_editor/`.

It is an [app-skeleton](../writing-an-app.md) GUI app. Its entry point hands its `App` implementation to
the skeleton's `run`, so the runtime owns the surface, the window, the input subscription, and the paint
loop (`userland/capsule_text_editor/src/main.rs:28`, `src/editor/app.rs:34`). The kernel spawns it under
service handle `app.text_editor` on service port 4726 with a reply port on 4727, and its capability mask
is `0x1819` (`userland/capsule_text_editor/Capsule.mk:11`).

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

The editor is an ordinary NONOS GUI application built on `nonos_app_skeleton`. The capsule supplies three
things and the runtime owns everything else: a manifest for a normal 500x320 window titled `Text Editor`
(`src/editor/manifest.rs:24`), an `on_event` that turns each key-down into an edit, a scroll, a
clipboard or file action, or a window close (`src/editor/event.rs:25`), and a `paint` that draws the
buffer into the surface the compositor presents (`src/editor/paint.rs:26`).

The document model is deliberately small. The whole document is one fixed array of `CAPACITY = 16384`
bytes with a `len`, and the "cursor" is implicit: text is always inserted at the end of the buffer and
Backspace always removes the last character, so there is no arbitrary insertion point to track
(`src/editor/state.rs:17`, `src/editor/insert.rs:20`, `src/editor/backspace.rs:20`). The buffer is
rendered with soft column wrapping computed from the window width, and the arrow, page, home, and end
keys move a scroll line over the wrapped view rather than an edit position (`src/editor/paint.rs:37`,
`src/editor/on_nav.rs:26`). The default path is `/notes.txt`, editable through an on-screen prompt before
an open or a save (`src/editor/state.rs:18`, `src/editor/path_prompt.rs:32`).

The editor holds no filesystem, network, or graphics-driver authority of its own. Open and save are IPC
calls to the vfs service, copy and paste are IPC calls to the clipboard service, and the save
notification is an IPC frame to the desktop shell; the far side holds the real authority in every case
(`src/editor/ctrl_open.rs:23`, `src/editor/ctrl_save.rs:22`, `src/editor/ctrl_copy.rs:21`,
`src/editor/notify.rs:30`).

## Identity

Everything the kernel and the service registry need to name and reach the editor comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `text-editor` | `Capsule.mk:1` |
| Service handle | `app.text_editor` | `Capsule.mk:2`, `src/userspace/capsule_text_editor/spawn.rs:31` |
| Namespace | `systems.nonos.app.text_editor` | `Capsule.mk:7` |
| Service endpoint | `service:4726:app.text_editor` | `Capsule.mk:8`, `spawn.rs:32` |
| Reply endpoint | `reply:4727:endpoint.app.text_editor.reply` | `Capsule.mk:9`, `spawn.rs:33`, `spawn.rs:34` |
| Capability mask | `0x1819` | `Capsule.mk:11` |
| Binary name | `text_editor` | `Capsule.mk:5` |
| Kernel mirror | `src/userspace/capsule_text_editor` | `Capsule.mk:12` |

The mask `0x1819` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x0001  CoreExec                bit()  1     types.rs:56
  0x0008  IPC                     bit()  8     types.rs:59
  0x0010  Memory                  bit() 16     types.rs:60
  0x0800  GraphicsDisplayQuery    bit() 2048   types.rs:67
  0x1000  GraphicsSurfaceCreate   bit() 4096   types.rs:68
  ------
  0x1819  = 1 + 8 + 16 + 2048 + 4096
```

The kernel spawn path requests exactly those five capabilities and no others
(`src/userspace/capsule_text_editor/spawn.rs:50`). There is no `Network` bit (4), no `FileSystem` bit
(64), and no hardware, driver, or DMA capability in the mask, which is the whole basis of the security
analysis below: the editor can create a surface, ask the display for its size, and speak IPC, and open,
save, copy, and paste are IPC calls to services that hold the real authority.

## User reference

Input arrives as key-down events. `on_event` ignores anything that is not a key-down; if a path prompt
is open it routes the key to the prompt handler; otherwise a Ctrl chord goes to `on_ctrl`, a navigation
key goes to `on_nav`, and everything else is editing (`src/editor/event.rs:25`). The key constants are
the app-skeleton codes shared with every capsule (`userland/app_skeleton/src/input/keys.rs`,
`src/input/modifiers.rs`).

### Editing

| Key | Action | Source |
|---|---|---|
| Printable 0x20..0x10FFFF | encode as UTF-8 and append at the end of the buffer | `event.rs:43`, `insert.rs:20` |
| Backspace | remove the last character, skipping UTF-8 continuation bytes | `event.rs:41`, `backspace.rs:20` |
| Enter | append a newline | `event.rs:42`, `insert.rs:20` |
| Esc | close the window | `event.rs:40` |

Insertion is bounded: `insert` appends only if the text fits within `CAPACITY`, and returns `false`
(dropping the input, no change) when the buffer is full, so a full document cannot overrun the array
(`src/editor/insert.rs:20`, `src/editor/state.rs:17`). Backspace walks back over UTF-8 continuation bytes
so one press removes a whole multi-byte character rather than half of one (`src/editor/backspace.rs:22`).
Any edit sets the status line to `edited` and scrolls the view to follow the end of the buffer
(`src/editor/event.rs:53`, `src/editor/follow_end.rs:21`). There is no in-text caret movement: the arrow
keys scroll, and there is no Left, Right, or Delete-at-cursor editing.

### Scrolling and navigation

Handled in `on_nav`, which returns a repaint for the keys it claims and passes the rest through to
editing (`src/editor/on_nav.rs:26`):

| Key | Action | Source |
|---|---|---|
| Up | scroll up one line | `on_nav.rs:28`, `scroll_up.rs:19` |
| Down | scroll down one line (clamped to the last page) | `on_nav.rs:29`, `scroll_down.rs:20` |
| Page Up | scroll up one screen | `on_nav.rs:30`, `scroll_up.rs:19` |
| Page Down | scroll down one screen (clamped) | `on_nav.rs:31`, `scroll_down.rs:20` |
| Home | jump to the top of the buffer | `on_nav.rs:32` |
| End | jump to the last screen of the buffer | `on_nav.rs:33`, `follow_end.rs:21` |

Scrolling is measured in wrapped visual lines, not raw newlines: the wrap width comes from the window
width clamped to 32..160 columns, the visible row count comes from the window height, and the scroll line
is clamped so the last page cannot scroll past the end (`src/editor/layout.rs:21`,
`src/editor/visible_rows.rs:19`, `src/editor/visual_lines.rs:17`, `src/editor/clamp_scroll.rs:21`,
`src/editor/max_scroll.rs:17`). Both the window size and the wrap are recomputed on every paint from the
current surface dimensions, so a resize reflows the text (`src/editor/paint.rs:27`).

### File open and save

Ctrl and its chords are handled in `on_ctrl`; both the upper and lower case key codes match so the chord
works regardless of the reported case (`src/editor/on_ctrl.rs:24`):

| Chord | Action | Source |
|---|---|---|
| Ctrl+O | open the path prompt for a load | `on_ctrl.rs:27`, `path_prompt.rs:23` |
| Ctrl+S | open the path prompt for a save | `on_ctrl.rs:28`, `path_prompt.rs:23` |
| Ctrl+C | copy the whole buffer to the clipboard | `on_ctrl.rs:26`, `ctrl_copy.rs:21` |
| Ctrl+V | paste clipboard text at the end of the buffer | `on_ctrl.rs:29`, `ctrl_paste.rs:22` |

Ctrl+O and Ctrl+S do not act immediately. Each opens a path prompt seeded with the current path (default
`/notes.txt`), sets a status line explaining the keys, and takes over input until the prompt is dismissed
(`src/editor/path_prompt.rs:23`). While the prompt is open (`src/editor/event.rs:29`):

| Key | Action | Source |
|---|---|---|
| ASCII graphic | append to the path (up to 255 bytes) | `path_prompt.rs:51` |
| Backspace | delete the last path character | `path_prompt.rs:39` |
| Enter | run the load (Open) or write (Save) against the typed path | `path_prompt.rs:44` |
| Esc | cancel the prompt, status becomes `cancelled` | `path_prompt.rs:35` |

On Enter for an Open, the editor resolves its own pid, stats the path, refuses a file larger than the
16 KiB buffer with `file too large`, reads it, and accepts it only if it is valid UTF-8 that fits;
otherwise the status is `file is not valid utf-8` or `open failed`, and a successful load reports
`opened` and scrolls to the end (`src/editor/ctrl_open.rs:23`). On Enter for a Save, the editor writes the
buffer to the path, notifies the desktop shell on success, and sets the status to `saved` or
`save failed` (`src/editor/ctrl_save.rs:22`).

Copy and paste do not use the prompt. Ctrl+C sends the whole buffer to the clipboard and reports
`copied /notes.txt` or `clipboard unavailable` (`src/editor/ctrl_copy.rs:21`). Ctrl+V pulls up to 512
bytes from the clipboard, accepts them only if they are valid UTF-8 and fit in the remaining buffer,
appends them, and reports `pasted into /notes.txt`, `paste rejected`, or `clipboard unavailable`
(`src/editor/ctrl_paste.rs:22`).

### Status line and window

The window is a fixed 500x320 Normal window that subscribes only to key-down input
(`src/editor/manifest.rs:19`, `src/editor/manifest.rs:33`). Every frame draws the title `text_editor`,
the current path, and the status line, then the wrapped buffer, then a `_` caret at the end of the text;
while a prompt is open a second `_` is drawn after the path to mark the edit point
(`src/editor/paint.rs:31`). The initial status line lists the four chords, `Ctrl-O open  Ctrl-S save
Ctrl-C copy  Ctrl-V paste`, and each action replaces it with its own message
(`src/editor/state_new.rs:30`).

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` calls the skeleton's `run(Editor::new)`, and the single
top-level module is `editor` (`src/main.rs:22`, `src/main.rs:28`). The `editor` module is one unit per
file: the `App` impl, the state model, the event router, the paint renderer, and one small file each for
insert, backspace, the Ctrl actions, the path prompt, the scroll math, and the layout and theme
constants (`src/editor/mod.rs:17`).

The model is a single `State` owned by the `Editor` (`src/editor/app.rs:24`). `State` holds the cached
owner pid, the fixed 16 KiB buffer and its length, the scroll line, the last computed visible-row and
wrap-column counts, the current status line, a 256-byte path with its length, an optional pending prompt,
and a cached desktop-shell port (`src/editor/state.rs:26`). It starts empty with the path preset to
`/notes.txt` and the four-chord hint as the status (`src/editor/state_new.rs:20`).

Lifecycle:

1. The kernel spawns the capsule at boot through the desktop fleet plan
   (`src/userspace/init/spawn_plan/apps_tools.rs:24`), which decodes the baked trust anchor and verifies
   the embedded ELF, id cert, manifest, and attestation trailer before registering `app.text_editor` on
   port 4726 (`src/userspace/capsule_text_editor/spawn.rs:37`, `spawn.rs:57`). On success the boot path
   logs `[APP-TEXT-EDITOR] capsule spawned` (`src/userspace/init/spawn_plan/apps_tools.rs:27`,
   `src/userspace/init/capsule_boot/run.rs:29`).
2. The skeleton `run` initialises the heap, waits for its service peers, builds the `Editor`, creates the
   window from the manifest, and drives the event, tick, and paint loop
   (`userland/app_skeleton/src/runner/entry.rs:31`).
3. Each key-down flows through `on_event` to the state; the return value tells the runtime to stay idle,
   repaint, or close (`src/editor/event.rs:25`, `userland/app_skeleton/src/app/event_outcome.rs:18`). Esc
   returns `Close`, which the runner turns into a window close (`src/editor/event.rs:40`,
   `userland/app_skeleton/src/runner/drain_ipc.rs:123`).
4. `paint` recomputes the visible rows and wrap columns from the surface, clamps the scroll, clears the
   background, and draws the header, path, status, the wrapped buffer row by row, and the caret; the frame
   lands in the shared surface the compositor presents (`src/editor/paint.rs:26`).

## Protocol and IPC

The editor exposes no application opcodes of its own beyond the `app.text_editor` service on port 4726
and the reply inbox on 4727 that the spawn path registers for it
(`src/userspace/capsule_text_editor/spawn.rs:31`). Everything it does that reaches outside the capsule is
an outbound IPC call to another service through the app skeleton's clients.

VFS, service `vfs_pool`, magic `0x4E4F5646`, through the skeleton's vfs client
(`userland/app_skeleton/src/clients/vfs/types.rs`):

```
  OP_OPEN     1     open a path                 types.rs:19
  OP_CLOSE    2     close a handle              types.rs:20
  OP_READ     3     read (Ctrl+O open)          types.rs:21
  OP_WRITE    4     write (Ctrl+S save)         types.rs:22
  OP_STAT     5     stat (open size precheck)   types.rs:23
```

Open calls `vfs::stat` to reject a file larger than the buffer, then `vfs::read_file`, which opens,
reads in chunks, and closes the handle (`src/editor/ctrl_open.rs:29`, `src/editor/ctrl_open.rs:35`,
`userland/app_skeleton/src/clients/vfs/read_file.rs:27`, `stat.rs:22`). Save calls `vfs::write_file`,
which opens with create, writes the buffer, and closes
(`src/editor/ctrl_save.rs:28`, `userland/app_skeleton/src/clients/vfs/write_file.rs:24`). Both pass the
editor's own pid, resolved once through the discovery client and cached
(`src/editor/resolve_owner_pid.rs:21`). The client surfaces its own error string on failure and the
editor maps it to a short status line.

Clipboard, service `clipboard`, magic NCLP `0x43424930`
(`userland/app_skeleton/src/wire/constants.rs:21`): `OP_COPY 0x0002` for Ctrl+C
(`userland/app_skeleton/src/clients/clipboard/copy.rs:22`) and `OP_PASTE 0x0003` for Ctrl+V
(`userland/app_skeleton/src/clients/clipboard/paste.rs:22`). Copy sends the whole buffer and reads a
status word; paste requests the content and reads back up to the caller's buffer, which the editor caps
at 512 bytes (`src/editor/ctrl_copy.rs:22`, `src/editor/ctrl_paste.rs:23`).

Desktop shell notify, service `desktop_shell`, magic NDSH `0x4E445348`, version 1, `OP_NOTIFY 0x0005`,
level info. After a successful save the editor looks up `desktop_shell` once, caches the port, builds a
20-byte header plus a bounded `saved <path>` body, and fires it with `mk_ipc_send`; the send is
best-effort and never blocks the save (`src/editor/notify.rs:22`, `src/editor/notify.rs:44`). If the
shell is not present the notify is skipped (`src/editor/notify.rs:34`).

Service discovery goes through the skeleton's `lookup_service`, used to resolve the editor's own pid and
the shell port (`userland/app_skeleton/src/discover/lookup_service.rs`,
`src/editor/resolve_owner_pid.rs:23`, `src/editor/notify.rs:32`).

## Security analysis

The editor reads and writes files and touches the clipboard, but its authority is exactly the app
envelope and nothing more. Its mask `0x1819` grants CoreExec, IPC, Memory, GraphicsDisplayQuery, and
GraphicsSurfaceCreate (`Capsule.mk:11`, `src/userspace/capsule_text_editor/spawn.rs:50`). There is no
Network bit, no FileSystem bit, and no hardware, driver, MMIO, or DMA capability. The editor cannot read
a block device, open a socket, or touch a device register on its own.

Open and save are IPC calls to the `vfs_pool` service, which holds the real filesystem authority and
applies its own checks; the editor marshals the path and the bytes and renders the reply
(`src/editor/ctrl_open.rs:35`, `src/editor/ctrl_save.rs:28`). Copy and paste are IPC calls to the
`clipboard` service (`src/editor/ctrl_copy.rs:22`, `src/editor/ctrl_paste.rs:24`). A bug in the editor's
parsing or rendering cannot escalate past what the vfs and the clipboard already permit for this pid,
because the editor never held more than the right to ask.

The editor also enforces its own conservative limits before it trusts any input. The buffer is a fixed
16 KiB array and `insert` refuses to append past it, so no edit or paste can overrun it
(`src/editor/insert.rs:20`, `src/editor/state.rs:17`). Open stats the file first and rejects anything
larger than the buffer, and accepts a loaded file only if it is valid UTF-8, so a hostile file cannot
smuggle non-text bytes into the buffer or overflow it (`src/editor/ctrl_open.rs:30`,
`src/editor/ctrl_open.rs:36`). Paste is capped at 512 bytes per press and is UTF-8 checked the same way
(`src/editor/ctrl_paste.rs:23`). The path is a 256-byte field and the prompt accepts only ASCII graphic
characters up to 255 bytes, so a path cannot grow without bound or carry control bytes
(`src/editor/state.rs:34`, `src/editor/path_prompt.rs:54`). The desktop-shell notify is best-effort and
carries only a bounded `saved <path>` string, and it is skipped entirely when the shell is absent
(`src/editor/notify.rs:34`, `src/editor/notify.rs:39`).

The capsule keeps no persistence and no state outside the process: the buffer, the path, and the scroll
position live only in `State` and disappear when the window closes (`src/editor/state.rs:26`). Isolation
from other capsules is the kernel's, not the editor's: it is a CPL 3 user binary that only speaks IPC and
its own surface, and it is verified and enrolled at spawn like every other capsule
(`src/userspace/capsule_text_editor/spawn.rs:40`).

## How to contribute

The source lives at `userland/capsule_text_editor/`. Everything is under `src/editor/`: the `App` impl in
`app.rs`, the model in `state.rs` and `state_new.rs`, the event router in `event.rs`, the Ctrl router in
`on_ctrl.rs`, the nav router in `on_nav.rs`, the renderer in `paint.rs`, and one file per action.

To add a key action or a chord:

1. Write the handler as its own file under `src/editor/`, following the existing shape: an editing helper
   is a method on `State` returning a `bool` for changed-or-not (as `insert` and `backspace` are), and a
   command is a `pub(super) fn action(state: &mut State) -> EventOutcome` (as `ctrl_copy` and
   `ctrl_paste` are). Register the module in `src/editor/mod.rs:17`.
2. Wire it into the right router. A plain key belongs in the `match` in `src/editor/event.rs:39`; a scroll
   or view key belongs in `src/editor/on_nav.rs:26`; a Ctrl chord belongs in `src/editor/on_ctrl.rs:24`,
   matching both the upper and lower case code the way the existing chords do. A key that should take over
   input until dismissed should open a prompt through `src/editor/path_prompt.rs:23`.
3. If the action touches a file, resolve the owner pid with `resolve_owner_pid` and go through the
   skeleton's vfs client rather than any raw syscall (`src/editor/resolve_owner_pid.rs:21`,
   `src/editor/ctrl_open.rs:17`). If it needs a new service, add a client under the app skeleton, not the
   editor.
4. Set a short static status line on the action's outcome so the result is visible, the way every existing
   action does (`src/editor/event.rs:53`, `src/editor/ctrl_save.rs:32`).

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
`nonos-mk/capsule.mk:182`, `nonos-mk/capsule.mk:261`, `nonos-mk/capsule.mk:263`, included through
`userland/capsule_text_editor/Capsule.mk:14`):

```
  make nonos-mk-text-editor              build the capsule ELF
  make nonos-mk-text-editor-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-text-editor-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-text-editor-keys   check the per-capsule signing keys exist
```

For a running desktop that includes the editor, `make nonos-mk-text-editor-prod` builds the full desktop
GUI image (`Makefile:1185`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every fallible path returns through a status line and an `EventOutcome`, and
UTF-8 decoding uses `from_utf8(...).is_ok()` and `unwrap_or("")` rather than an unwrapping panic; the
release profile is `panic = "abort"`, `Cargo.toml:26`); modular files, one unit per file, with `mod.rs`
used only for module declarations and the single `pub use`; and the AGPL header at the top of every
source file, matching the header on every existing module (`src/main.rs:1`).

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[APP-TEXT-EDITOR] capsule spawned` (tag `APP-TEXT-EDITOR`, message `capsule spawned`) from the boot log
(`src/userspace/init/spawn_plan/apps_tools.rs:27`, `src/userspace/init/capsule_boot/run.rs:29`,
`src/sys/boot_log/output.rs:33`). An absent line means the capsule never started, usually a signature,
manifest, or capability failure; the error path prints an `[ERROR]` line instead
(`src/userspace/init/capsule_boot/run.rs:32`, `src/sys/boot_log/output.rs:49`).

Failure modes and where to look:

- Editor opens but no key does anything. The window subscribes only to key-down and `on_event` drops
  everything else, so if keys are dead the input path into the app (compositor, wm, input_router) is the
  suspect, not the editor (`src/editor/manifest.rs:33`, `src/editor/event.rs:26`).
- Open fails. The status line names the stage: `open failed` when the pid cannot be resolved or the vfs
  read errors, `file too large` when the stat exceeds the 16 KiB buffer, and `file is not valid utf-8`
  when the bytes are not text (`src/editor/ctrl_open.rs:25`, `src/editor/ctrl_open.rs:31`,
  `src/editor/ctrl_open.rs:43`). A `save failed` on Ctrl+S is the same split between pid resolution and the
  vfs write (`src/editor/ctrl_save.rs:24`, `src/editor/ctrl_save.rs:32`).
- Copy or paste does nothing. `clipboard unavailable` means the clipboard service did not answer,
  `paste rejected` means the clipboard content was not valid UTF-8 or did not fit the remaining buffer
  (`src/editor/ctrl_copy.rs:24`, `src/editor/ctrl_paste.rs:32`, `src/editor/ctrl_paste.rs:35`).
- A save succeeds but no desktop notification appears. The notify is best-effort and is skipped when
  `desktop_shell` is not registered, so a missing toast is expected without the shell and never blocks the
  save (`src/editor/notify.rs:31`, `src/editor/notify.rs:34`).
- Text looks clipped or wrapped oddly after a resize. Wrap columns and visible rows are recomputed from the
  surface every paint and clamped to 32..160 columns, so an unexpected wrap points at the reported surface
  size (`src/editor/layout.rs:21`, `src/editor/visible_rows.rs:19`, `src/editor/paint.rs:27`).

## Source map

```
  src/main.rs                        _start -> run(Editor::new)
  src/editor/mod.rs                  module list and the Editor re-export
  src/editor/app.rs                  Editor: the App impl (manifest, on_event, paint)
  src/editor/state.rs                State: buffer, len, scroll, path, prompt, ports
  src/editor/state_new.rs            State::new: empty buffer, /notes.txt, status hint
  src/editor/event.rs                on_event: prompt / ctrl / nav / edit routing
  src/editor/insert.rs               append to the buffer, bounded by CAPACITY
  src/editor/backspace.rs            remove the last character, UTF-8 aware
  src/editor/on_nav.rs               arrow, page, home, end scrolling
  src/editor/scroll_up.rs            scroll up
  src/editor/scroll_down.rs          scroll down (clamped)
  src/editor/clamp_scroll.rs         clamp the scroll line to the last page
  src/editor/max_scroll.rs           last scrollable line
  src/editor/follow_end.rs           scroll to the end of the buffer
  src/editor/visual_lines.rs         count wrapped lines
  src/editor/end_position.rs         caret line and column at the buffer end
  src/editor/visible_rows.rs         visible rows from the window height
  src/editor/layout.rs               wrap columns and the glyph metrics
  src/editor/on_ctrl.rs              Ctrl chord router (O, S, C, V)
  src/editor/ctrl_open.rs            open a file through the vfs client
  src/editor/ctrl_save.rs            save the buffer through the vfs client
  src/editor/ctrl_copy.rs            copy the buffer to the clipboard
  src/editor/ctrl_paste.rs           paste clipboard text into the buffer
  src/editor/path_prompt.rs          the open/save path prompt input mode
  src/editor/resolve_owner_pid.rs    cache the editor's own pid for vfs calls
  src/editor/notify.rs               NDSH save notification to desktop_shell
  src/editor/paint.rs                the frame renderer
  src/editor/theme.rs                the colour constants
  src/editor/manifest.rs             the window manifest (500x320, key-down only)
  Capsule.mk                         slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_text_editor/ the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/apps_tools.rs   the desktop-fleet spawn entry
  userland/app_skeleton/src/clients/ the vfs, clipboard, and discovery clients the editor calls
  nonos-mk/capsule.mk                the generated nonos-mk-text-editor[-sign|-verify] targets
```

Every reference above is verified against those trees.

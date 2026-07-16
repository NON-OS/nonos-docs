# capsule_file_manager (full reference)

`capsule_file_manager` is the desktop file browser in the NONOS tree: a GUI window that lists a
directory, previews files, and performs the ordinary create, rename, delete, copy, move, duplicate,
and permission operations against the virtual filesystem. It does none of that itself. Every file
operation is an IPC call to the `vfs_pool` service, which holds the real authority. This is the
exhaustive reference.

It is an [app-skeleton](../writing-an-app.md) GUI app. The kernel spawns it under service handle
`app.file_manager` on service port 4724 with a reply port on 4725, and its capability mask is `0x1819`
(`userland/capsule_file_manager/Capsule.mk:11`). The source is `userland/capsule_file_manager/`.

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

The file manager is an ordinary NONOS GUI application. Its entry point hands its `App` implementation
to the skeleton's `run`, so the runtime owns the surface, the window, the input subscription, and the
paint loop, and the file manager supplies three things: a manifest for a normal window, an `on_event`
that turns keystrokes and clicks into browsing and file actions, and a `paint` that draws the current
directory listing (or a preview, or the help overlay) into the surface
(`userland/capsule_file_manager/src/main.rs:28`, `src/fm/app.rs:37`).

The model is a single `State`: the current directory prefix, the full listing, the filtered and sorted
view of it, the cursor, the scroll offset, an optional open preview, a status line, the current input
mode, the pending prompt text, the live filter, the sort mode, a checkbox selection set, and a
copy/cut clipboard (`src/fm/state.rs:50`). The app has five interaction modes: browsing, an incremental
filter, a help overlay, a text prompt (new file, mkdir, rename, delete confirmation), and a file
preview (`src/fm/state.rs:26`). Directory contents, file sizes, and modification times come from the
`vfs_pool` service; the capsule itself cannot read a block device.

## Identity

Everything the kernel and the service registry need to name and reach the file manager comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `file-manager` | `Capsule.mk:1` |
| Service handle | `app.file_manager` | `Capsule.mk:2`, `src/userspace/capsule_file_manager/spawn.rs:31` |
| Namespace | `systems.nonos.app.file_manager` | `Capsule.mk:7` |
| Service endpoint | `service:4724:app.file_manager` | `Capsule.mk:8`, `spawn.rs:32` |
| Reply endpoint | `reply:4725:endpoint.app.file_manager.reply` | `Capsule.mk:9`, `spawn.rs:33` |
| Capability mask | `0x1819` | `Capsule.mk:11` |
| Binary name | `file_manager` | `Capsule.mk:5` |
| Kernel mirror | `src/userspace/capsule_file_manager` | `Capsule.mk:12` |

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
(`src/userspace/capsule_file_manager/spawn.rs:49`). There is no `Network` bit (4), and crucially no
`FileSystem` bit (64), no hardware, driver, MMIO, or DMA capability in the mask. That is the basis of
the security analysis below: the file manager can create a surface, ask the display for its size, and
speak IPC, and every action that appears to touch a file is really an IPC call to the `vfs_pool`
service, which holds the real authority.

## User reference

Input arrives as key-down events and absolute pointer button-down events. `on_event` first runs
`select_row` to move the cursor to a clicked row while browsing, ignores anything that is neither a
button-down nor a key-down, then routes to the mode-specific handler if the app is in a non-browse
mode, and otherwise treats the event as a browse key (`src/fm/event_dispatch.rs:24`). A pointer
button-down while browsing is folded into the same path as pressing Enter: the row under the pointer is
selected by `select_row`, then the click is dispatched as `KEY_ENTER`, so a single click opens the
entry it lands on (`src/fm/event_dispatch.rs:25`, `:32`, `src/fm/event_mouse.rs:22`).

### Browse mode

While browsing, `on_browse_key` first offers the code to `run_action` (the file actions) and then to
`open_mode` (the mode switches), and only if neither claims it does it fall through to the navigation
keys (`src/fm/event_browse.rs:30`).

Navigation and opening (`src/fm/event_browse.rs`):

| Key | Action | Handler |
|---|---|---|
| Up / `k` | move the cursor up one row | `event_browse.rs:37`, `:40`, `event_move.rs:22` |
| Down / `j` | move the cursor down one row | `event_browse.rs:38`, `:39`, `event_move.rs:22` |
| Enter / Right / `l` | open: enter a directory, or preview a file | `event_browse.rs:35`, `:42`, `event_open.rs:23` |
| Backspace / Left / `h` | go up to the parent directory | `event_browse.rs:36`, `:41`, `event_parent.rs:22` |
| left mouse click on a row | select that row and open it | `event_dispatch.rs:25`, `:32`, `event_mouse.rs:22` |
| Esc | close the window | `event_browse.rs:34` |

Opening a directory replaces the prefix and refreshes the listing; opening a file reads it through the
vfs and switches to preview mode (`src/fm/event_open.rs:24`). Going up truncates the prefix to its
parent and refreshes; at `/` it does nothing (`src/fm/event_parent.rs:23`). The cursor move is clamped
to the entry count and keeps the scroll window positioned so the cursor stays visible
(`src/fm/event_move.rs:27`, `src/fm/scroll.rs:23`).

File actions, dispatched in `run_action` (`src/fm/event_actions.rs:28`):

| Key | Action | Handler |
|---|---|---|
| space | check or uncheck the entry under the cursor | `event_actions.rs:30`, `selection.rs:20` |
| `a` | select every entry in the current view | `event_actions.rs:31`, `selection_select_all.rs:19` |
| `c` | copy the acting set to the clipboard (p to paste) | `event_actions.rs:32`, `clipboard.rs:34` |
| `x` | cut the acting set to the clipboard (p to paste) | `event_actions.rs:33`, `clipboard.rs:34` |
| `p` | paste the clipboard into the current directory | `event_actions.rs:34`, `clipboard_paste.rs:22` |
| `o` | duplicate the acting set in place under a `(copy)` name | `event_actions.rs:35`, `duplicate.rs:28` |
| `u` | toggle the acting set between writable and read-only | `event_actions.rs:36`, `perms.rs:29` |
| `s` | cycle the sort mode name -> size -> date -> type | `event_actions.rs:37`, `sort_next.rs:20` |

The acting set is the checkbox selection when any entries are checked, and otherwise just the entry
under the cursor, so every batch action works whether or not a selection is active
(`src/fm/selection_acting.rs:23`). Copy and cut record the acting paths into the clipboard, marking the
cut flag, and clear the checkbox selection (`src/fm/clipboard.rs:40`). Paste renames each clip on a cut
or copies it on a copy, into the current directory under its original base name; a cut clipboard is
emptied after the paste (`src/fm/clipboard_paste.rs:34`). Duplicate copies each acting entry into the
current directory under a non-colliding `name (copy).ext` (`src/fm/duplicate.rs:39`, `:52`). The
read-only toggle flips each entry between mode `0o644` and `0o444` based on its current writable flag
(`src/fm/perms.rs:24`, `:38`).

Prompts, opened from browse mode by `start_prompt` (`src/fm/prompt_start.rs:22`):

| Key | Action | Handler |
|---|---|---|
| `n` | prompt for a name and create an empty file | `event_browse.rs:43`, `prompt_run_op.rs:29` |
| `m` | prompt for a name and create a directory | `event_browse.rs:44`, `prompt_run_op.rs:32` |
| `r` | prompt for a new name and rename the cursor entry | `event_browse.rs:45`, `prompt_run_op.rs:33` |
| `d` | prompt for a `y` confirmation and delete the cursor entry | `event_browse.rs:46`, `prompt_run_op.rs:38` |

Mode switches, handled in `open_mode` (`src/fm/event_modes.rs:21`):

| Key | Action | Handler |
|---|---|---|
| `/` | enter incremental filter mode | `event_modes.rs:23` |
| `?` | open the full-window keybinding help | `event_modes.rs:27` |

### Filter mode

Filter mode is a live incremental search over the listing. Each ascii-graphic keystroke (up to 48
characters) appends to the filter and rebuilds the view immediately, keeping only entries whose name
contains the filter case-insensitively; Backspace pops a character and rebuilds; Esc clears the filter
and returns to browsing; Enter keeps the filter and returns to browsing
(`src/fm/filter.rs:24`, `src/fm/view.rs:28`).

### Prompt mode

A prompt collects a single line of text. Each ascii-graphic keystroke (up to 64 characters) appends to
the input, Backspace pops, Esc cancels back to browse mode, and Enter commits the operation for the
prompt kind (`src/fm/prompt.rs:24`). Commit rejects an empty name for everything but delete, joins the
name to the current prefix to form the target, runs the vfs operation, refreshes, and shows the
resulting status (`src/fm/prompt_commit.rs:23`). Delete only proceeds when the typed text is exactly
`y`, and it refuses directories with `dirs not supported` (`src/fm/prompt_run_op.rs:39`, `:44`).

### Preview mode

Opening a file reads up to 256 KiB through the vfs and renders it as scrollable lines: text files show
their content, binary files show a hexdump, and the header notes when the file was longer than what is
shown (`src/fm/preview.rs:29`, `:44`). In preview mode Up and Down scroll the visible window and Esc
returns to browsing (`src/fm/preview_key.rs:24`). A failed read leaves browse mode with a `read failed`
status (`src/fm/preview.rs:60`).

### Help mode

`?` opens a full-window reference of the keybindings; any key dismisses it back to browsing
(`src/fm/help.rs:27`, `:58`). The help text is the authoritative in-app cheat sheet and matches the
handlers above.

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` calls the skeleton's `run(FileManager::new)`
(`src/main.rs:27`). The single top-level module is `fm`, which is split one unit per file: the `App`
implementation (`app.rs`), the manifest, the event handlers (`event_*`), the file actions (clipboard,
duplicate, perms, selection), the vfs refresh, the view/filter/sort, the preview, the prompts, and the
paint pass (`paint_*`) (`src/fm/mod.rs:17`).

`FileManager` wraps one `State`, and its constructor does an initial `refresh` so the window shows the
root directory as soon as it appears (`src/fm/app.rs:30`). Both `on_event` and `paint` re-run `refresh`
if the owner pid has not been resolved yet or the last refresh reported `vfs unavailable`, so the
listing self-heals once the vfs service comes up after the window (`src/fm/app.rs:43`, `:50`).

Lifecycle:

1. The kernel spawns the capsule at boot through the desktop fleet plan
   (`src/userspace/init/spawn_plan/apps.rs:98`), which verifies the embedded ELF, cert, manifest, and
   attestation, registers `app.file_manager` on port 4724, and on success logs
   `[APP-FILE-MANAGER] capsule spawned` (`src/userspace/capsule_file_manager/spawn.rs:37`,
   `src/userspace/init/spawn_plan/apps.rs:101`, `src/userspace/init/capsule_boot/run.rs:29`).
2. The skeleton `run` creates the window from the manifest: a 360x260 Normal window titled
   `File Manager`, subscribing to key-down, button-down, and absolute-pointer input
   (`src/fm/manifest.rs:26`).
3. Each event flows in through `on_event`. Browsing keys move the cursor and open entries; actions
   mutate the store through the vfs and refresh; mode keys switch into filter, help, prompt, or
   preview, each with its own handler (`src/fm/event_dispatch.rs:24`).
4. `paint` projects the current mode into the surface. Help and preview take over the whole window;
   otherwise it draws the title, the current path, a sort/filter header, the listing rows, and the
   footer status and counter (`src/fm/paint.rs:28`). The listing is a scrolling window of
   `LIST_VISIBLE = 7` rows at `ROW_HEIGHT = 22` starting at y 64, each row drawn with its name, a
   right-aligned human size, and a right-aligned modified time, with the cursor row highlighted
   (`src/fm/layout.rs:19`, `src/fm/paint_rows.rs:31`). The frame lands in the shared surface the
   compositor presents.

The refresh path resolves the owner pid once by looking up `app.file_manager` in the registry, lists
the directory prefix through the vfs, builds the entry list, fills per-file metadata by `stat` for
directories of at most 128 entries, then rebuilds the filtered and sorted view
(`src/fm/refresh.rs:25`, `src/fm/refresh_meta.rs:21`).

## Protocol and IPC

The file manager exposes no application opcodes of its own beyond what the app skeleton registers for
it (the `app.file_manager` service on port 4724 and the reply inbox on 4725,
`src/userspace/capsule_file_manager/spawn.rs`). Everything it does that reaches outside the capsule is
an outbound IPC call to the `vfs_pool` service, through the skeleton's vfs client. Each call frames a
request with `magic | op | request_id | body`, sends it with `mk_ipc_call`, and reads back a status
word (`userland/app_skeleton/src/clients/vfs/call.rs:21`).

VFS, service `vfs_pool`, magic `0x4E4F5646`
(`userland/app_skeleton/src/clients/vfs/types.rs:17`, `:18`). The ops the file manager actually uses:

```
  OP_OPEN     1     open a path (read_file, write_file)   types.rs:19
  OP_CLOSE    2     close a handle (read_file, write_file) types.rs:20
  OP_READ     3     read a file (preview)                 types.rs:21
  OP_WRITE    4     write a file (new file)               types.rs:22
  OP_STAT     5     stat (size, mtime, mode for listing)  types.rs:23
  OP_LIST     6     list a directory prefix (refresh)     types.rs:24
  OP_MKDIR    8     mkdir (m prompt)                      types.rs:25
  OP_UNLINK   9     unlink (delete, cut source path)      types.rs:26
  OP_RENAME   10    rename (r prompt, cut paste)          types.rs:27
  OP_COPY     11    copy (copy paste, duplicate)          types.rs:28
  OP_CHMOD    15    chmod (u read-only toggle)            types.rs:32
```

Mapped to actions:

- `list_paths` for the directory refresh, over `OP_LIST`; it returns the length-prefixed path list the
  view is built from (`src/fm/refresh.rs:30`, `list_paths.rs:29`).
- `stat_full` for per-file size, mtime, and mode, over `OP_STAT`
  (`src/fm/refresh_meta.rs:32`, `stat_full.rs:35`).
- `read_file` for the preview, over `OP_OPEN`/`OP_READ`/`OP_CLOSE`, capped at 256 KiB
  (`src/fm/preview.rs:45`, `read_file.rs:35`).
- `write_file` for a new empty file, over `OP_OPEN`/`OP_WRITE`/`OP_CLOSE`
  (`src/fm/prompt_run_op.rs:30`, `write_file.rs:36`).
- `mkdir` for a new directory (`src/fm/prompt_run_op.rs:32`, `mkdir.rs:32`).
- `rename` for rename and for the cut-paste move (`src/fm/prompt_run_op.rs:35`,
  `src/fm/clipboard_paste.rs:35`, `rename.rs:34`).
- `unlink` for delete (`src/fm/prompt_run_op.rs:46`, `unlink.rs:32`).
- `copy` for copy-paste and for duplicate, with a recursive flag for directories
  (`src/fm/clipboard_paste.rs:37`, `src/fm/duplicate.rs:41`, `copy.rs:38`).
- `chmod` for the read-only toggle (`src/fm/perms.rs:40`, `chmod.rs:36`).

Service discovery uses `lookup_service` to resolve `app.file_manager` to the owner pid that the vfs
calls are keyed on (`src/fm/refresh.rs:28`, `userland/app_skeleton/src/discover/lookup_service.rs`). A
vfs reply carries a status word; a non-zero status or a short reply surfaces as the client's error
string, which the file manager turns into a status line (`call.rs:30`, `list_paths.rs:30`).

## Security analysis

The file manager looks like it owns the filesystem, but its authority is exactly the app envelope and
nothing more. Its mask `0x1819` grants CoreExec, IPC, Memory, GraphicsDisplayQuery, and
GraphicsSurfaceCreate (`Capsule.mk:11`, `src/userspace/capsule_file_manager/spawn.rs:49`). There is no
Network bit, no FileSystem bit, and no hardware, driver, MMIO, or DMA capability. The capsule cannot
read a block device, open a socket, or touch a device register on its own.

Every operation that appears to touch a file is an IPC call to the `vfs_pool` service, which holds the
real authority and applies its own checks. Listing, stat, read, write, mkdir, unlink, rename, copy, and
chmod are all requests through the skeleton's vfs client (`src/fm/refresh.rs`, `src/fm/prompt_run_op.rs`,
`src/fm/clipboard_paste.rs`, `src/fm/duplicate.rs`, `src/fm/perms.rs`). The file manager marshals the
argument bytes and renders the reply; the service on the far side decides whether the operation is
allowed for this pid. A bug in path handling or in a batch action cannot escalate past what the vfs
already permits, because the file manager never held more than the right to ask.

The calls are keyed on the owner pid the capsule resolves for its own service handle, not on an
arbitrary caller, so the file manager operates under its own identity
(`src/fm/refresh.rs:28`). There is no launch path: the file manager has no installer client and no
process-spawn capability, so opening a file only reads it into a preview; it never executes anything.
Deletion is deliberately conservative: it refuses directories outright (`prompt_run_op.rs:44`) and
requires an explicit `y` confirmation (`prompt_run_op.rs:39`). Reads are bounded at 256 KiB so opening a
large file stays finite (`src/fm/preview.rs:29`), and metadata stat is skipped for directories over 128
entries so a huge listing does not stall (`src/fm/refresh_meta.rs:24`).

Isolation from other capsules is the kernel's, not the file manager's: it is a CPL 3 user binary that
only speaks IPC and its own surface, and it is verified and enrolled at spawn like every other capsule
(`src/userspace/capsule_file_manager/spawn.rs`). Its clipboard and selection live entirely in its own
`State` (`src/fm/state.rs:50`) and are never shared.

## How to contribute

The source lives at `userland/capsule_file_manager/`. Everything is under `src/fm/`, one unit per file:
`app.rs` and `main.rs` for the entry and `App` impl, `manifest.rs` for the window, the `event_*` files
for input, the action files (`clipboard.rs`, `clipboard_paste.rs`, `duplicate.rs`, `perms.rs`,
`selection*.rs`), `refresh.rs`/`refresh_meta.rs` for the vfs listing, `view*.rs`/`filter.rs`/`sort_*.rs`
for the visible list, `preview*.rs` for the file preview, `prompt*.rs` for the text prompts, and the
`paint_*` files for the renderer.

To add a browse action (a new single-key operation over the selection or cursor):

1. Write the operation as its own module under `src/fm/`, taking `&mut State`. Get the acting set with
   `crate::fm::selection_acting::acting` if it should work on the selection or the cursor, run the vfs
   client call, then `refresh` and set a status. Follow `duplicate.rs` or `perms.rs` as the template.
2. Wire the key into the match in `src/fm/event_actions.rs:28` and return `EventOutcome::Repaint`.
3. Add a line to the in-app help table in `src/fm/help.rs:27` so the `?` overlay documents it.

To add a prompt-driven operation (one that collects a name first), add a `PromptKind` variant in
`src/fm/state.rs:34`, a browse key that calls `start_prompt` in `src/fm/event_browse.rs`, and the
matching arm in `src/fm/prompt_run_op.rs:28`. To reach a vfs op the client does not expose yet, add the
thin wrapper under `userland/app_skeleton/src/clients/vfs/` alongside the existing `mkdir.rs`,
`rename.rs`, and `chmod.rs`, using the opcode from
`userland/app_skeleton/src/clients/vfs/types.rs`.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_file_manager/Capsule.mk:14`):

```
  make nonos-mk-file-manager             build the capsule ELF
  make nonos-mk-file-manager-sign        produce the id cert, manifest, and attestation trailer
  make nonos-mk-file-manager-verify      verify the signed artifacts against the trust anchor
  make nonos-mk-check-file-manager-keys  check the per-capsule signing keys exist
```

For a running desktop that includes the file manager, `make nonos-mk-file-manager-prod` builds the full
desktop GUI image (`Makefile:1166`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every action reports errors as a status line and swallows the client's
`Result`, never a panic; the release profile is `panic = "abort"`, `Cargo.toml`); modular files, one
unit per file, with `mod.rs` used only for re-exports; and the AGPL header at the top of every source
file, matching the header on every existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[APP-FILE-MANAGER] capsule spawned` (tag `APP-FILE-MANAGER`, message `capsule spawned`) from the boot
log (`src/userspace/init/spawn_plan/apps.rs:101`, `src/userspace/init/capsule_boot/run.rs:29`,
`src/sys/boot_log/output.rs:33`). An absent line means the capsule never started, usually a signature,
manifest, or capability failure; the error path prints an `[ERROR]` line instead
(`src/userspace/init/capsule_boot/run.rs:32`, `src/sys/boot_log/output.rs:49`).

Failure modes and where to look:

- The window shows `vfs unavailable`. The refresh could not resolve the owner pid or the vfs did not
  answer, and the listing is empty (`src/fm/refresh.rs:44`). Because both `on_event` and `paint` retry
  the refresh while the status is `vfs unavailable`, the listing recovers on its own once `vfs_pool`
  comes up; a persistent `vfs unavailable` points at the vfs service, not the file manager
  (`src/fm/app.rs:43`).
- The window shows `refresh deferred`. A later refresh failed but a prior listing exists, so the old
  entries are kept rather than blanked (`src/fm/refresh.rs:47`). This is a transient vfs hiccup, not a
  crash.
- `empty directory` versus `no matches`. An empty directory shows `empty directory`; a non-empty
  directory with an active filter that matches nothing shows `no matches`
  (`src/fm/paint_rows.rs:33`, `src/fm/refresh.rs:35`). Clearing the filter with Esc distinguishes the
  two.
- An action reports `... some failed`. Paste, duplicate, and chmod run per-entry and set a
  `paste: some failed`, `duplicate: some failed`, or `chmod: some failed` status if any single vfs call
  returned an error, while the ones that succeeded still applied
  (`src/fm/clipboard_paste.rs:45`, `src/fm/duplicate.rs:47`, `src/fm/perms.rs:46`). The split is between
  the file manager and the vfs: the capsule only issues the request and reports the aggregate.
- Delete does nothing. Delete only proceeds on an exact `y`; anything else leaves `not deleted`, and a
  directory target is refused with `dirs not supported` (`src/fm/prompt_run_op.rs:39`, `:44`).
- A file will not preview. `read failed` means the vfs `read_file` returned an error; the preview is
  cleared and browsing continues (`src/fm/preview.rs:60`). Files over 256 KiB are truncated, not
  refused, and the info bar marks them (`src/fm/preview.rs:53`).

## Source map

```
  src/main.rs                          _start -> run(FileManager::new)
  src/fm/mod.rs                        module list; re-exports FileManager
  src/fm/app.rs                        FileManager: App impl (manifest, on_event, paint)
  src/fm/manifest.rs                   window: 360x260 Normal, key/button/pointer input mask
  src/fm/state.rs                      State and the Mode / PromptKind / SortMode enums
  src/fm/state_new.rs                  State::new (root prefix, browse mode, name sort)
  src/fm/event_dispatch.rs             on_event: mouse row select, mode route, browse fall-through
  src/fm/event_mouse.rs                select_row: click -> cursor row
  src/fm/event_mode.rs                 route: dispatch by non-browse mode
  src/fm/event_modes.rs                open_mode: '/' filter, '?' help
  src/fm/event_browse.rs               on_browse_key: navigation, open, parent, prompts
  src/fm/event_actions.rs              run_action: space a c x p o u s
  src/fm/event_open.rs, event_parent.rs, event_move.rs   open / up / cursor move
  src/fm/selection*.rs                 checkbox selection, select-all, acting set
  src/fm/clipboard.rs, clipboard_paste.rs   copy/cut yank and paste
  src/fm/duplicate.rs, perms.rs        in-place duplicate, read-only toggle
  src/fm/prompt*.rs                    prompt input, commit, and the vfs op per PromptKind
  src/fm/filter.rs, view*.rs, sort_*.rs   incremental filter, filtered/sorted view
  src/fm/preview*.rs                   file preview (text / hexdump, scroll)
  src/fm/refresh.rs, refresh_meta.rs   vfs listing and per-file stat
  src/fm/paint*.rs, layout.rs, theme.rs   the renderer (title, header, rows, footer, preview, help)
  Capsule.mk                           slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_file_manager/  the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/apps.rs   the desktop-fleet spawn entry
  userland/app_skeleton/src/clients/vfs/  the vfs client the actions call (types.rs holds the opcodes)
  nonos-mk/capsule.mk                  the generated nonos-mk-file-manager[-sign|-verify] targets
```

Every reference above is verified against those trees.

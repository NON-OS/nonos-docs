# Debugging

This page covers the runtime: the boot marker that says the capsule ran, and the status-line failure
modes and where to look when the listing, an action, or the preview misbehaves. The status line the file
manager sets is the primary diagnostic; it is drawn by the footer (see [rendering.md](rendering.md)).

## Confirm the capsule ran

On a successful boot the kernel prints `[APP-FILE-MANAGER] capsule spawned`. The spawn helper passes the
tag `APP-FILE-MANAGER` and the message `capsule spawned` to `boot_log::ok`, which prints
`[<tag>] <msg>` (`src/userspace/init/spawn_plan/apps.rs:101`, `src/userspace/init/capsule_boot/run.rs:29`,
`src/sys/boot_log/output.rs:33`). An absent line means the capsule never started, usually a signature,
manifest, or capability failure; the error path prints an `[ERROR]` line instead
(`src/userspace/init/capsule_boot/run.rs:32`, `src/sys/boot_log/output.rs:49`).

## Listing failure modes

- The window shows `vfs unavailable`. The refresh could not resolve the owner pid or the vfs did not
  answer, and there was no prior listing to keep (`src/fm/refresh.rs:45`). Because both `on_event` and
  `paint` retry the refresh while the status is `vfs unavailable`, the listing recovers on its own once
  `vfs_pool` comes up; a persistent `vfs unavailable` points at the vfs service, not the file manager
  (`src/fm/app.rs:43`, `:50`).
- The window shows `refresh deferred`. A later refresh failed but a prior listing exists, so the old
  entries are kept rather than blanked (`src/fm/refresh.rs:47`). This is a transient vfs hiccup, not a
  crash.
- `empty directory` versus `no matches`. An empty directory shows `empty directory`; a non-empty
  directory with an active filter that matches nothing shows `no matches`
  (`src/fm/paint_rows.rs:33`, `src/fm/refresh.rs:35`). Clearing the filter with Esc distinguishes the
  two.

## Action failure modes

- An action reports `... some failed`. Paste, duplicate, and the read-only toggle run per-entry and set
  `paste: some failed`, `duplicate: some failed`, or `chmod: some failed` if any single vfs call
  returned an error, while the ones that succeeded still applied (`src/fm/clipboard_paste.rs:45`,
  `src/fm/duplicate.rs:47`, `src/fm/perms.rs:46`). The split is between the file manager and the vfs: the
  capsule only issues the request and reports the aggregate.
- `nothing to yank`, `nothing to duplicate`, `nothing selected`, `clipboard empty`. A batch action ran
  with an empty acting set or an empty clipboard (`src/fm/clipboard.rs:37`, `src/fm/duplicate.rs:31`,
  `src/fm/perms.rs:32`, `src/fm/clipboard_paste.rs:23`). Check the cursor is on an entry or a selection
  is active.
- Delete does nothing. Delete only proceeds on an exact `y`; anything else leaves `not deleted`, and a
  directory target is refused with `dirs not supported` (`src/fm/prompt_run_op.rs:39`, `:44`).
- A prompt reports `empty name`. Commit rejects an empty name for everything but delete
  (`src/fm/prompt_commit.rs:26`).

## Preview failure modes

- A file will not preview. `read failed` means the vfs `read_file` returned an error; the preview is
  cleared and browsing continues (`src/fm/preview.rs:60`).
- A large file is truncated, not refused. Reads are bounded at 256 KiB; the info bar marks a truncated
  file with `(truncated)` (`src/fm/preview.rs:29`, `src/fm/preview_info.rs:25`).

## Where the authority sits

The file manager holds no FileSystem capability; every operation is an IPC call to `vfs_pool`, which
applies its own checks (see [README](README.md#identity)). When an action fails, the question is whether
the capsule sent a malformed request or the service refused it. The file manager only marshals the
argument bytes and renders the reply status; the decision is on the far side.

## Source map

Everything here is drawn from the boot path (`src/userspace/init/spawn_plan/apps.rs`,
`src/userspace/init/capsule_boot/run.rs`, `src/sys/boot_log/output.rs`) and the status-setting units
under `userland/capsule_file_manager/src/fm/` (`refresh.rs`, `paint_rows.rs`, `clipboard.rs`,
`clipboard_paste.rs`, `duplicate.rs`, `perms.rs`, `prompt_run_op.rs`, `prompt_commit.rs`, `preview.rs`,
`preview_info.rs`, `app.rs`). Every reference above is verified against those trees.

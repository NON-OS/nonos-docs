# The File Manager Capsule

The file manager is the desktop file browser in the NONOS tree: a signed userland capsule that draws
its own window, lists a directory, previews files, and performs the ordinary create, rename, delete,
copy, move, duplicate, and permission operations. It does none of that itself. Every file operation is
a capability-checked IPC call to the `vfs_pool` service, which holds the real authority. The source is
one top-level module, `fm`, split one unit per file, and this documentation groups those units into the
concerns they form so a page can be read beside the folder it describes.

## Identity

| Field | Value | Source |
|-------|-------|--------|
| Slug | `file-manager` | `userland/capsule_file_manager/Capsule.mk:1` |
| Service handle | `app.file_manager` | `Capsule.mk:2`, `src/userspace/capsule_file_manager/spawn.rs:31` |
| Namespace | `systems.nonos.app.file_manager` | `Capsule.mk:7` |
| Service endpoint | `service:4724:app.file_manager` | `Capsule.mk:8`, `spawn.rs:32` |
| Reply endpoint | `reply:4725:endpoint.app.file_manager.reply` | `Capsule.mk:9`, `spawn.rs:33` |
| Binary name | `file_manager` | `Capsule.mk:5` |
| Kernel mirror | `src/userspace/capsule_file_manager` | `Capsule.mk:12` |
| Capability mask | `0x1859` | `Capsule.mk:12` |

The mask `0x1859` decomposes into exactly six bits, checked against `src/capabilities/types/defs.rs`:

| Bit | Value | Grants |
|-----|-------|--------|
| CoreExec | `0x0001` | run as a process |
| IPC | `0x0008` | send and receive on its endpoints |
| Memory | `0x0010` | map its own heap and stack |
| FileSystem | `0x0040` | the filesystem-capability gate its vfs calls are checked against |
| GraphicsDisplayQuery | `0x0800` | ask the compositor for the display geometry |
| GraphicsSurfaceCreate | `0x1000` | create the window surface it draws into |

`0x1859 = 0x0001 + 0x0008 + 0x0010 + 0x0040 + 0x0800 + 0x1000`. The kernel spawn path requests exactly
those six capabilities and no others (`src/userspace/capsule_file_manager/spawn.rs:49`). There is no
Network bit (`0x0004`), and no hardware, driver, MMIO, or DMA capability. The capsule holds `FileSystem`
as the capability gate for its vfs requests, but it cannot read a block device, open a socket, or touch a
device register on its own: every action that touches a file is an IPC call to the `vfs_pool` service,
which holds the storage authority and enforces the path checks.

## The pillars

The source under `userland/capsule_file_manager/src/` is one module, `fm`, with `_start` handing
`FileManager::new` to the app skeleton's `run` (`src/main.rs:28`). The units group into five concerns.
An event comes in through the input layer, may run an action or open a prompt, both of which mutate the
model through the vfs, and the model is what the renderer draws. The preview is its own full-window
takeover.

```
  input     ->   actions    ->   listing    ->   rendering
  event_*        clipboard       refresh /       paint_* /
  routing        duplicate       entries /       layout /
                 perms /         view /          theme /
                 selection /     filter /        manifest /
                 prompt          sort            help
                                    |
                                 preview
                                 preview_*
```

| Page | Mirrors | What it covers |
|------|---------|----------------|
| [input.md](input.md) | `event_dispatch.rs`, `event_mode.rs`, `event_modes.rs`, `event_browse.rs`, `event_actions.rs`, `event_mouse.rs`, `event_move.rs`, `event_open.rs`, `event_parent.rs`, `scroll.rs` | The event router, the mouse row select, the browse keys, navigation and opening, and the mode switches. |
| [actions.md](actions.md) | `selection*.rs`, `clipboard.rs`, `clipboard_paste.rs`, `duplicate.rs`, `perms.rs`, `prompt*.rs` | The file operations: the checkbox selection and acting set, copy and cut, paste, in-place duplicate, the read-only toggle, and the name prompts (new file, mkdir, rename, delete). |
| [listing.md](listing.md) | `state.rs`, `state_new.rs`, `entries.rs`, `refresh.rs`, `refresh_meta.rs`, `view*.rs`, `filter.rs`, `sort_*.rs` | The `State` model, the vfs directory refresh and per-file stat, and the filtered and sorted view. |
| [preview.md](preview.md) | `preview.rs`, `preview_hex.rs`, `preview_text.rs`, `preview_is_binary.rs`, `preview_key.rs`, `preview_paint.rs`, `preview_clip.rs`, `preview_info.rs` | The file preview: reading a file through the vfs, the text and hexdump renderers, scrolling, and the truncation notice. |
| [rendering.md](rendering.md) | `manifest.rs`, `paint.rs`, `paint_*.rs`, `layout.rs`, `theme.rs`, `help.rs`, plus the file-decoration units | The window manifest, the paint pass (title, header, rows, footer), the row geometry, the palette, and the help overlay. |
| [contributing.md](contributing.md) | the whole tree | Where to work, how to add an action or a prompt, the build and sign steps, and the code standards. |
| [debugging.md](debugging.md) | runtime | The boot marker, the status-line failure modes, and where to look when the listing, an action, or the preview misbehaves. |

The vfs client the actions and the listing call lives outside the capsule, in the app skeleton at
`userland/app_skeleton/src/clients/vfs/`; the opcodes it uses are named on both [actions.md](actions.md)
and [listing.md](listing.md) and defined in that client's `types.rs`.

## Lifecycle

The file manager is spawned through [verified spawn](../../../security/capsules-and-trust.md): its
signature and attestation are checked, its requested capabilities are held against its manifest ceiling,
and only then is its ELF mapped (`src/userspace/capsule_file_manager/spawn.rs:37`). It registers
`app.file_manager` at port 4724, and the skeleton `run` creates the window from the manifest and enters
the input-driven paint loop. `FileManager::new` does an initial `refresh` so the window shows the root
directory as soon as it appears (`src/fm/app.rs:32`). A successful spawn prints
`[APP-FILE-MANAGER] capsule spawned` on the boot log
(`src/userspace/init/spawn_plan/apps.rs:101`); the [debugging](debugging.md) page covers the runtime
markers.

## Source map

Everything here is drawn from `userland/capsule_file_manager/` (the capsule source and its `Capsule.mk`),
`src/capabilities/types/defs.rs` (the capability bits), the kernel spawn mirror under
`src/userspace/capsule_file_manager/`, and the shared vfs client under
`userland/app_skeleton/src/clients/vfs/`. Every reference above is verified against those trees.

# capsule_settings (full reference)

`capsule_settings` is the system control panel for NONOS: a GUI window that reads and writes the machine's
policy store. It presents three tabs of settings (Display, Network, Security), and each row is a live
control backed by a single field in the policy service. Toggling a boolean, nudging a slider, cycling an
enum, or editing a hostname is a real IPC write to the `policy` service, which is the one place in the
system that owns persistent user and kernel policy. This is the exhaustive reference.

It is an [app-skeleton](../writing-an-app.md) GUI app. The kernel spawns it under service handle
`app.settings` on service port 4728 with a reply port on 4729, and its capability mask is `0x1819`
(`userland/capsule_settings/Capsule.mk:11`). The source is `userland/capsule_settings/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Settings reference](#settings-reference)
- [Interaction and keybindings](#interaction-and-keybindings)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

Settings is an ordinary NONOS GUI application. Its entry point hands its `App` implementation to the
skeleton's `run`, so the runtime owns the surface, the window, the input subscription, and the paint
loop, and the capsule supplies three things: a manifest for a normal window, an `on_event` that turns
keystrokes and pointer clicks into policy writes, and a `paint` that draws the current values
(`userland/capsule_settings/src/main.rs:28`, `src/settings/app.rs:56`).

The model is a flat list of policy fields grouped into three categories. On startup the capsule looks up
the `policy` service, reads every field once to fill an in-memory cache, and then draws the active
category as a scrollable list of labelled rows. Every field carries a kind (bool, u8, i8, or string) and,
for enum and numeric fields, a range, all of which come from the shared `nonos_policy_proto` crate rather
than being hard-coded in the capsule (`userland/policy_proto/src/field_kind.rs:20`,
`src/field_max.rs:20`). When the user changes a control, the capsule sends a single `OP_SET` to the policy
service; only if the service accepts the write does the capsule update its cached value and show
`updated` in the status bar (`src/settings/event/commit_bool.rs:27`).

Settings holds no policy of its own and stores nothing. It is a viewer and an editor for the policy
store, and the store, not the capsule, decides whether any given write is allowed.

## Identity

Everything the kernel and the service registry need to name and reach the settings capsule comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `settings` | `Capsule.mk:1` |
| Service handle | `app.settings` | `Capsule.mk:2`, `src/userspace/capsule_settings/spawn.rs:30` |
| Namespace | `systems.nonos.app.settings` | `Capsule.mk:7` |
| Service endpoint | `service:4728:app.settings` | `Capsule.mk:8`, `spawn.rs:31` |
| Reply endpoint | `reply:4729:endpoint.app.settings.reply` | `Capsule.mk:9`, `spawn.rs:32`, `spawn.rs:33` |
| Capability mask | `0x1819` | `Capsule.mk:11` |
| Binary name | `settings` | `Capsule.mk:5` |
| Kernel mirror | `src/userspace/capsule_settings` | `Capsule.mk:12` |

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
(`src/userspace/capsule_settings/spawn.rs:49`). There is no `Network` bit (4), no `FileSystem` bit (64),
and no hardware, driver, or DMA capability in the mask. Settings can create a surface, ask the display
for its size, and speak IPC, and nothing else. Its power to change system policy is not a capability it
holds; it comes entirely from the policy service recognising its service name, which is the whole basis of
the security analysis below.

Settings is one of exactly two capsules the policy service will accept a write from. The policy server
holds a two-entry allow list, `app.settings` and `app.setup_wizard`, and rejects an `OP_SET` from any
other sender with `E_ACCES` (`userland/capsule_policy/src/server/handle_set.rs:23`, `:41`). The setup
wizard writes the same store during first-boot provisioning; settings is the ongoing editor.

## Settings reference

Every row the user sees is one `Field` from `nonos_policy_proto`. The capsule's schema lists all
thirty-seven fields it knows about (`src/settings/schema/all_fields.rs:19`) and assigns each to one of the
three tabs (`src/settings/schema/visible_for.rs:53`). A field's kind decides how its row behaves
(`userland/policy_proto/src/field_kind.rs:20`):

- `KIND_BOOL` (1): a toggle. Space, Enter, or a click on the value flips it and writes the new bool
  (`src/settings/event/toggle_or_inc.rs:30`).
- `KIND_U8` (2): a numeric or enum value. Left and Right adjust it by one, clamped to the field's max
  (`src/settings/event/adjust_u8.rs:26`). Fields with an enum table render as `< Label >  [n/total]`
  (`src/settings/paint/paint_value_enum.rs:25`); the rest render as a decimal.
- `KIND_I8` (3): a signed value. Left and Right adjust it, clamped to -12..=14
  (`src/settings/event/adjust_i8.rs:25`, `:33`). Only the timezone field uses this.
- `KIND_STR` (4): free text. Enter opens an inline editor; typing appends, Backspace deletes, Enter
  commits the write, Esc cancels (`src/settings/event/on_event_editing.rs:24`).

The label shown for each row comes from `label_of` (`userland/policy_proto/src/field_label.rs:19`); the
values a bool or numeric or enum field can take, and the range, come from `kind_of`, `max_of`, and the
enum tables. The three tab labels are `Display`, `Network`, and `Security`
(`src/settings/paint/paint_tabs.rs:25`), which map to the `Category::User`, `Category::Identity`, and
`Category::Kernel` values internally.

Note that a field's tab is set by the schema's `visible_for` grouping, not by its numeric category. A few
identity-category fields (anonymous mode, Nym routing) live under the Network tab because that is where
`visible_for` places them (`src/settings/schema/visible_for.rs:32`). The `SoundEnabled`,
`NotificationsEnabled`, `KeyboardLayout`, `Timezone`, `Language`, `DeveloperMode`, `SystemKeysGenerated`,
`AutoWipe`, `WifiAutoconnect`, and several kernel fields are declared in `ALL_FIELDS` and are hydrated and
writable, but are not placed on any of the three tabs by the current `visible_for`, so they do not appear
as rows; the tables below mark which fields are visible.

### Display tab (`Category::User`)

Rows from `DISPLAY_FIELDS` (`src/settings/schema/visible_for.rs:19`), in order:

| Row label | Field | Kind | Control and range | What it writes |
|---|---|---|---|---|
| Display brightness | `Brightness` `0x0101` | u8 | Left/Right, 0..=100 | `OP_SET` u8 to `Brightness` (`field_max.rs:25`) |
| Pointer speed | `MouseSensitivity` `0x0102` | u8 | Left/Right, 0..=4 | `OP_SET` u8 to `MouseSensitivity` (`field_max.rs:26`) |
| Cursor size | `CursorSize` `0x0116` | u8 enum | Left/Right cycles Small, Normal, Large, Huge (0..=3) | `OP_SET` u8 index (`cursor_size_labels.rs:17`) |
| High contrast mode | `HighContrast` `0x0111` | bool | Space/Enter toggles | `OP_SET` bool |
| Text size | `FontSize` `0x0112` | u8 enum | Left/Right cycles Tiny, Small, Normal, Large, Huge (0..=4) | `OP_SET` u8 index (`font_size_labels.rs:17`) |
| Color theme | `Theme` `0x0106` | u8 enum | Left/Right cycles Aurora, Slate, Nord, Dracula, Solar, Mono, Forest, Sunset (0..=7) | `OP_SET` u8 index (`theme_labels.rs:17`) |
| Wallpaper | `Wallpaper` `0x0117` | u8 enum | Left/Right cycles the 62 wallpaper names (0..=61) | `OP_SET` u8 index (`wallpaper_labels.rs:17`) |
| Screen blank timeout (min) | `ScreenTimeout` `0x010A` | u8 | Left/Right, 0..=240 | `OP_SET` u8 to `ScreenTimeout` (`field_max.rs:27`) |
| UI animations | `AnimationsEnabled` `0x0115` | bool | Space/Enter toggles | `OP_SET` bool |
| 24-hour clock | `ClockFormat24` `0x0118` | bool | Space/Enter toggles | `OP_SET` bool |

The wallpaper enum has 62 entries named `Field Focus 01..13`, `Hardware Aesthetic 01..14`, `Network
Topology 01..19`, and `Special Variant 1a..15` (`wallpaper_labels.rs:17`), and its selected index is what
the wallpaper capsule reads back to choose the background.

### Network tab (`Category::Identity`)

Rows from `NETWORK_FIELDS` (`src/settings/schema/visible_for.rs:32`), in order:

| Row label | Field | Kind | Control | What it writes |
|---|---|---|---|---|
| Wi-Fi auto-connect | `WifiAutoconnect` `0x0114` | bool | Space/Enter toggles | `OP_SET` bool |
| Anonymous mode | `AnonymousMode` `0x0104` | bool | Space/Enter toggles | `OP_SET` bool |
| Nym routing | `NymEnabled` `0x0105` | bool | Space/Enter toggles | `OP_SET` bool |
| Hostname | `Hostname` `0x0301` | string | Enter opens editor, up to 63 chars, `[A-Za-z0-9._-]` only | `OP_SET` str to `Hostname` |
| Domain name | `DomainName` `0x0302` | string | Enter opens editor, up to 63 chars, `[A-Za-z0-9._-]` only | `OP_SET` str to `DomainName` |

The `Wi-Fi auto-connect` row is the settings side of Wi-Fi: it toggles the `WifiAutoconnect` policy bit
that the network stack reads to decide whether to associate on boot. It is a single boolean and nothing
more; the network capsule owns the actual scanning, association, and credential handling. For how Wi-Fi
brings a link up, see the [networking subsystem](../../subsystems/networking/README.md).

Hostname and domain name are the only free-text controls. The capsule accepts only ASCII letters,
digits, `.`, `-`, and `_` while editing (`src/settings/event/push_text_char.rs:33`), and the policy
server independently re-validates the bytes with the same character set before storing them
(`userland/capsule_policy/src/store/str_validate.rs:17`), so an invalid string is rejected on both sides.
The string cap is 63 bytes (`userland/policy_proto/src/limits.rs:17`).

### Security tab (`Category::Kernel`)

Rows from `SECURITY_FIELDS` (`src/settings/schema/visible_for.rs:40`), in order:

| Row label | Field | Kind | Control | What it writes |
|---|---|---|---|---|
| Auto-lock after idle (min) | `AutoLockTimeout` `0x0113` | u8 | Left/Right, 0..=240 | `OP_SET` u8 (`field_max.rs:28`) |
| Auto-wipe on shutdown | `AutoWipe` `0x0108` | bool | Space/Enter toggles | `OP_SET` bool |
| Hardware crypto offload | `HardwareCrypto` `0x010D` | bool | Space/Enter toggles | `OP_SET` bool |
| Zero-knowledge attestation | `ZkAttestation` `0x010E` | bool | Space/Enter toggles | `OP_SET` bool |
| Developer mode | `DeveloperMode` `0x010C` | bool | Space/Enter toggles | `OP_SET` bool |
| Kernel ASLR | `KernelAslr` `0x0201` | bool | Space/Enter toggles | `OP_SET` bool |
| NX bit enforcement | `KernelNxBit` `0x0203` | bool | Space/Enter toggles | `OP_SET` bool |
| SMEP (supervisor exec prevention) | `KernelSmep` `0x0204` | bool | Space/Enter toggles | `OP_SET` bool |
| SMAP (supervisor access prevention) | `KernelSmap` `0x0205` | bool | Space/Enter toggles | `OP_SET` bool |
| Seccomp syscall filter | `KernelSeccomp` `0x020C` | bool | Space/Enter toggles | `OP_SET` bool |

Every security row is a boolean written through the same gated `OP_SET` path as any other field; setting
one of these tells the policy store the desired kernel posture, and the store applies it downstream
through its push path (`userland/capsule_policy/src/push/mod.rs:17`). The row labels come straight from
`field_label.rs` (for example `field_label.rs:45` for Kernel ASLR).

### Fields present but not on a tab

`ALL_FIELDS` also declares `SoundEnabled` `0x0103`, `KeyboardLayout` `0x0107`, `Timezone` `0x0109`,
`Language` `0x010B`, `SystemKeysGenerated` `0x010F`, `NotificationsEnabled` `0x0110`, and the kernel
fields `KernelStackGuard`, `KernelDebug`, `KernelSerial`, `KernelWatchdog`, `KernelPreempt`,
`KernelHugepages`, `KernelIommu` (`all_fields.rs:19`). These are hydrated into the cache at startup and
the policy protocol supports getting and setting them, but the current `visible_for` grouping does not put
them on any of the three tabs, so they are not shown as editable rows. The `KeyboardLayout`, `Language`,
and `Timezone` fields carry full metadata (enum tables and the i8 range) and would render correctly if
added to a tab (`keyboard_layout_labels.rs:17`, `language_labels.rs:17`, `field_kind.rs:32`).

## Interaction and keybindings

Input arrives as key-down events and pointer button-down events. `on_event` routes a `ButtonDown` to the
pointer handler, ignores anything that is not a key-down, then splits keyboard handling by whether a
string edit is in progress (`src/settings/event/on_event.rs:25`).

Browsing (not editing), handled in `on_event_browsing` (`src/settings/event/on_event_browsing.rs:30`):

| Key | Action | Source |
|---|---|---|
| Up | move the cursor up one row (wraps) | `on_event_browsing.rs:34`, `state/cursor_up.rs` |
| Down | move the cursor down one row (wraps) | `on_event_browsing.rs:35`, `state/cursor_down.rs:21` |
| Left | decrement the current numeric/enum field, or nudge down | `on_event_browsing.rs:36`, `event/adjust.rs:24` |
| Right | increment the current numeric/enum field, or nudge up | `on_event_browsing.rs:37`, `event/adjust.rs:24` |
| Space or Enter | toggle a bool, cycle an enum, or open the string editor | `on_event_browsing.rs:38`, `event/toggle_or_inc.rs:24` |
| Tab or `]` | switch to the next category tab | `on_event_browsing.rs:33`, `:40`, `event/next_category.rs:22` |
| `[` | switch to the previous category tab | `on_event_browsing.rs:39`, `event/prev_category.rs` |
| Esc | close the window | `on_event_browsing.rs:32` |

Space and Enter both call `toggle_or_inc`: a bool field flips and writes immediately; a string field opens
the inline editor; anything else (a numeric or enum field) increments by one
(`src/settings/event/toggle_or_inc.rs:29`). Left and Right go through `adjust`, which dispatches to the
u8 or i8 adjuster by the field's kind and does nothing for bool or string fields
(`src/settings/event/adjust.rs:29`).

Editing a string field, handled in `on_event_editing` (`src/settings/event/on_event_editing.rs:24`):

| Key | Action | Source |
|---|---|---|
| Printable `[A-Za-z0-9._-]` | append to the edit buffer | `on_event_editing.rs:38`, `event/push_text_char.rs:19` |
| Backspace | delete the last character | `on_event_editing.rs:34` |
| Enter | commit the edited string as an `OP_SET` write | `on_event_editing.rs:30`, `event/commit_string.rs:24` |
| Esc | cancel the edit and discard the buffer | `on_event_editing.rs:26`, `state/edit_cancel.rs:20` |

Pointer input, handled in `on_pointer` (`src/settings/event/on_pointer.rs:34`): a click in the tab strip
selects the tab under the pointer (`on_pointer.rs:39`); a click on a row selects that row, and then a
click on the left third of the value area decrements, a click on the right edge increments, and a click
in the middle toggles or cycles (`on_pointer.rs:61`). Clicks outside the body do nothing
(`on_pointer.rs:35`, `:50`).

The status bar at the bottom shows a keybinding hint when idle, `updated` in green after a successful
write, an error message in red after a rejected one, and `policy unavailable; showing static defaults`
when the policy service could not be reached (`src/settings/paint/paint_status.rs:28`).

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` calls the skeleton's `run(Settings::new)`
(`src/main.rs:28`). The module tree under `src/settings/` is `app` (the `App` impl), `event` (input and
writes), `ipc` (the policy client), `paint` (the renderer), `schema` (the field list and tab grouping),
`state` (the model), plus `manifest` and `theme` (`src/settings/mod.rs:17`).

The model is a single `State`: the resolved policy port and a ready flag, the active category, a
per-category cursor and scroll position, a fixed array of cached field values, an editing flag and edit
buffer, and a status line (`src/settings/state/state.rs:27`). The value array has one slot per field in
`ALL_FIELDS`, indexed by `slot_of` (`src/settings/state/slot_of.rs:21`), and every slot starts `Unknown`
until hydration fills it (`src/settings/state/new.rs:24`).

Lifecycle:

1. The kernel spawns the capsule at boot through the desktop fleet plan, which verifies the embedded ELF,
   cert, manifest, and attestation, registers `app.settings` on port 4728, and logs `[APP-SETTINGS]
   capsule spawned` (`src/userspace/capsule_settings/spawn.rs:36`,
   `src/userspace/init/spawn_plan/apps_tools.rs:39`, `src/userspace/init/spawn_plan/boot.rs:26`).
2. `Settings::new` looks up the `policy` service and, if found, records its port and marks policy ready
   (`src/settings/app.rs:31`). The skeleton then creates the window from the manifest (a 760x520 Normal
   window titled `NONOS Settings`, subscribing to key-down input) (`src/settings/manifest.rs:24`).
3. On the first event or paint after the port is known, `ensure_ready` hydrates: it reads every field once
   with `OP_GET`, stores each returned value, and yields between reads so the read burst does not starve
   the scheduler (`src/settings/app.rs:42`, `src/settings/ipc/hydrate.rs:24`). Hydration runs once.
4. Each key-down or click flows through `on_event` to the browsing, editing, or pointer handler, which may
   send an `OP_SET` and update the cache and status on success (`src/settings/event/on_event.rs:25`).
5. `paint` clears the surface, draws the header and the three tabs, draws the visible rows of the active
   category with the cursor highlighted, draws a scroll indicator, and draws the status bar
   (`src/settings/paint/paint.rs:31`). Only rows that fit are drawn; the visible-row count is derived from
   the window height (`src/settings/paint/visible_rows.rs:21`), and the cursor is kept on screen by
   `track_scroll` (`src/settings/state/track_scroll.rs:21`).

## Protocol and IPC

The settings capsule speaks the policy protocol defined in `nonos_policy_proto`. The service is `policy`
on port 4108 with reply port 4109 (`userland/policy_proto/src/service.rs:17`). Each message is a 12-byte
header (`op`, `field`, `kind`, a status word, and a payload length) followed by the payload
(`userland/policy_proto/src/hdr.rs:17`). There are two operations: `OP_GET` (0x0001) and `OP_SET`
(0x0002) (`userland/policy_proto/src/ops.rs:17`).

Resolving the service: `lookup_policy_port` calls `mk_service_lookup` for the name `policy` and fails with
`NotFound` if the port comes back zero (`src/settings/ipc/lookup.rs:24`). Until the lookup succeeds the
status bar shows `policy unavailable` and the rows display static defaults.

Reading: `op_get` sends an `OP_GET` with an empty payload and decodes the reply by kind (a bool byte, a
u8, an i8, or a string), checking that the reply's kind matches what was requested
(`src/settings/ipc/op_get.rs:26`). Hydration calls this once per field
(`src/settings/ipc/hydrate.rs:29`).

Writing: the four setters `op_set_bool`, `op_set_u8`, `op_set_i8`, and `op_set_str` each send an `OP_SET`
with the encoded value (`src/settings/ipc/op_set_bool.rs:22`, `op_set_str.rs:22`). All of them go through
one `call` function, which frames the header, sends the request with a 500 ms timeout, checks the reply
length and status, and returns an error if the status is not `E_OK`
(`src/settings/ipc/call.rs:28`, `src/settings/ipc/timeout.rs:17`). The full error set is timeout, short
reply, bad header, kind mismatch, send-failed, not-found, and a status code the server returned
(`src/settings/ipc/error.rs:17`); each maps to a specific status-bar message
(`src/settings/event/report.rs:22`).

Notify: after any successful `OP_SET`, `call` sends the desktop shell an `NDSH` notify frame (magic
`0x4E44_5348`, op `0x0005`, level info, body `settings applied`) so the shell can surface a toast; the
send is best-effort and its failure is ignored (`src/settings/ipc/call.rs:73`,
`src/settings/ipc/notify_shell.rs:32`). This is the only service settings talks to other than `policy`.

On the server side, an `OP_SET` is dispatched by `handle_set::dispatch`, which first checks the sender is
a trusted setter and otherwise replies `E_ACCES`, then routes by kind to the matching store handler
(`userland/capsule_policy/src/server/handle_set.rs:40`). The sender pid used in that check is the
attested pid the kernel delivers with the message, read out of `mk_ipc_recv_from`
(`userland/capsule_policy/src/server/recv.rs:21`).

## Security analysis

Settings can rewrite kernel security policy, the hostname, and the whole user preference set, so it is a
high-value capsule, but its authority is narrow and its trust is explicit.

Its capability mask is `0x1819`: CoreExec, IPC, Memory, GraphicsDisplayQuery, and GraphicsSurfaceCreate
(`Capsule.mk:11`, `src/userspace/capsule_settings/spawn.rs:49`). There is no Network bit, no FileSystem
bit, and no hardware, driver, MMIO, or DMA capability. Settings cannot touch a device, open a socket, or
read a block store. It changes system behaviour only by sending `OP_SET` messages to the `policy`
service, which owns the store and applies the effects.

The gate that authorises those writes lives in the policy service, not in settings. When the policy
server receives an `OP_SET`, it takes the sender pid the kernel attested on the message
(`userland/capsule_policy/src/server/recv.rs:21`), and calls `is_trusted_setter`, which looks up the pids
currently registered for the service names `app.settings` and `app.setup_wizard` and returns true only if
the sender matches one of them (`userland/capsule_policy/src/server/handle_set.rs:23`, `:36`). Any other
sender gets `E_ACCES` and the store is untouched (`handle_set.rs:41`). So the right to write policy is not
a token settings carries and could leak; it is the policy service recognising, per message, that the
caller is the settings pid (or the setup wizard pid) that the registry says owns `app.settings`. A capsule
that forged the wire format but ran under a different pid would still be rejected, because the check is on
the attested sender, not on anything in the payload.

What settings still cannot do, even as a trusted setter:

- It cannot bypass the store's own validation. A bad-length or out-of-range value is rejected by the
  matching store handler with `E_INVAL` or `E_BAD_LEN`, and settings only shows `policy rejected`; it has
  no way to force the write (`userland/capsule_policy/src/server/handlers/set_str.rs:24`, `:29`,
  `src/settings/event/report.rs:29`).
- It cannot write a hostname or domain name outside `[A-Za-z0-9._-]`. The capsule filters the keystrokes
  and the server independently re-validates the bytes, so the two checks must agree
  (`src/settings/event/push_text_char.rs:33`, `userland/capsule_policy/src/store/str_validate.rs:17`).
- It cannot invent fields. It can only get and set the `Field` values the shared protocol enumerates
  (`userland/policy_proto/src/field.rs:19`); an unknown field id does not decode
  (`userland/policy_proto/src/field_decode.rs`).
- It cannot spawn code, present another capsule's surface, or reach the network. Its mask forbids all
  three, and its only outbound calls are to `policy` and, best-effort, to `desktop_shell` for a toast.

Isolation from other capsules is the kernel's, not the capsule's: settings is a CPL 3 user binary that
only speaks IPC and its own surface, and it is verified and enrolled at spawn like every other capsule
(`src/userspace/capsule_settings/spawn.rs:36`).

## How to contribute

The source lives at `userland/capsule_settings/`. The `App` impl is in `src/settings/app.rs`, the input
and write handlers under `src/settings/event/`, the policy client under `src/settings/ipc/`, the renderer
under `src/settings/paint/`, the field list and tab grouping under `src/settings/schema/`, and the model
under `src/settings/state/`. The field metadata (names, labels, kinds, ranges, enum tables) lives in the
shared `userland/policy_proto/` crate, which both the capsule and the policy service depend on.

To add a new setting:

1. Add the field to the shared protocol. Give it an id in the right category range in
   `userland/policy_proto/src/field.rs:19` (`0x01xx` user, `0x02xx` kernel, `0x03xx` identity), a label in
   `field_label.rs:19`, a kind in `field_kind.rs:20`, and, if it is numeric, a max in `field_max.rs:20`.
   For an enum add a labels table and wire it into `enum_table.rs:25`. The policy service must know how to
   store it, so add the matching store field and handler under `userland/capsule_policy/src/store/`.
2. Register it in the capsule schema. Add the field to `ALL_FIELDS`
   (`src/settings/schema/all_fields.rs:19`) so it is hydrated and gets a cache slot, and add it to the
   right group in `visible_for` (`src/settings/schema/visible_for.rs:19`) so it shows up as a row on a
   tab. No new event or paint code is needed: the row's behaviour follows from its kind.

To add a new control kind (beyond bool, u8, i8, and string) you would extend `kind.rs`, the `adjust` and
`toggle_or_inc` dispatch (`src/settings/event/adjust.rs:29`, `toggle_or_inc.rs:29`), and the paint value
renderers (`src/settings/paint/`), but the four existing kinds cover every field today.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_settings/Capsule.mk:14`):

```
  make nonos-mk-settings                build the capsule ELF
  make nonos-mk-settings-sign           produce the id cert, manifest, and attestation trailer
  make nonos-mk-settings-verify         verify the signed artifacts against the trust anchor
  make nonos-mk-check-settings-keys     check the per-capsule signing keys exist
```

For a running desktop that includes settings, `make nonos-mk-settings-prod` builds the full desktop GUI
image (it maps to the `nonos-mk-desktop-gui-prod` target, `Makefile:1186`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every write returns errors as a status line and leaves the cache unchanged,
never a panic; the release profile is `panic = "abort"`, `Cargo.toml`); modular files, one unit per file,
with `mod.rs` used only for re-exports; and the AGPL header at the top of every source file, matching the
header on every existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints `[APP-SETTINGS]
capsule spawned` (tag `APP-SETTINGS`, message `capsule spawned`) from the boot log
(`src/userspace/init/spawn_plan/boot.rs:26`, `src/sys/boot_log/output.rs:32`). An absent line means the
capsule never started, usually a signature, manifest, or capability failure; the error path prints an
error line instead (`src/userspace/init/spawn_plan/boot.rs:29`).

Failure modes and where to look:

- Window opens but every row shows `...` and the status bar reads `policy unavailable; showing static
  defaults`. The capsule could not resolve the `policy` service, so nothing hydrated
  (`src/settings/ipc/lookup.rs:32`, `src/settings/paint/paint_status.rs:28`). Confirm the policy capsule
  is registered; the values remain blank until the lookup succeeds and `ensure_ready` runs hydration
  (`src/settings/app.rs:42`).
- A change shows `policy rejected` (red status). The write reached the policy service and the service
  returned a non-`E_OK` status, which the capsule maps to `policy rejected`
  (`src/settings/event/report.rs:29`, `src/settings/ipc/call.rs:70`). The usual cause is an out-of-range
  numeric value or an invalid hostname character, rejected by the store handler with `E_INVAL` or
  `E_BAD_LEN` (`userland/capsule_policy/src/server/handlers/set_str.rs:24`,
  `userland/policy_proto/src/errno.rs:18`).
- A change shows `policy timeout` or `ipc send failed`. The request did not get a reply within 500 ms, or
  the send itself failed (`src/settings/ipc/timeout.rs:17`, `src/settings/ipc/error.rs:20`,
  `report.rs:24`). This points at the policy service being wedged or gone, not at the settings UI.
- Every write is refused even though the row accepts the keypress. If the policy service returns `E_ACCES`
  the status shows `policy rejected`; that is the trusted-setter gate denying the write because the
  sender pid is not the registered `app.settings` pid (`userland/capsule_policy/src/server/handle_set.rs:41`).
  This normally only happens if the registry entry for `app.settings` is missing or stale.
- The hostname or domain editor drops keystrokes. Only `[A-Za-z0-9._-]` are accepted while editing; any
  other printable byte is silently ignored (`src/settings/event/push_text_char.rs:33`).

## Source map

```
  src/main.rs                              _start -> run(Settings::new)
  src/settings/app.rs                      App impl: lookup, hydrate-once, dispatch to event/paint
  src/settings/manifest.rs                 760x520 Normal window, key-down subscription
  src/settings/schema/all_fields.rs        the full field list (cache slots + hydration order)
  src/settings/schema/visible_for.rs       the three tab groups (Display/Network/Security)
  src/settings/state/                      State: category, cursor, scroll, value cache, edit buffer, status
  src/settings/event/on_event.rs           key vs pointer, browsing vs editing split
  src/settings/event/{adjust,toggle_or_inc}.rs   Left/Right adjust and Space/Enter toggle/cycle/edit
  src/settings/event/{commit_bool,commit_string,adjust_u8,adjust_i8}.rs   the write paths
  src/settings/event/{push_text_char,report}.rs   string editing filter and status messages
  src/settings/ipc/lookup.rs               resolve the policy service
  src/settings/ipc/{hydrate,op_get}.rs     read every field once
  src/settings/ipc/{call,op_set_bool,op_set_u8,op_set_i8,op_set_str}.rs   the OP_SET write path
  src/settings/ipc/notify_shell.rs         best-effort NDSH toast after a successful write
  src/settings/paint/                      header, tabs, rows, scroll indicator, status bar
  Capsule.mk                               slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_settings/          the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/apps_tools.rs   the desktop-fleet spawn entry
  userland/policy_proto/                   shared Field enum, labels, kinds, ranges, enum tables, wire header
  userland/capsule_policy/src/server/handle_set.rs   the trusted-setter gate (app.settings, app.setup_wizard)
  nonos-mk/capsule.mk                      the generated nonos-mk-settings[-sign|-verify] targets
```

The policy store this capsule edits is documented in the [policy capsule reference](policy.md), and the
Wi-Fi link the auto-connect toggle governs is documented in the
[networking subsystem](../../subsystems/networking/README.md).

Every reference above is verified against those trees.

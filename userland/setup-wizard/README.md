# capsule_setup_wizard (full reference)

`capsule_setup_wizard` is the first-boot setup wizard: the full-screen guided flow that runs after the
core desktop services are up but before the desktop fleet, and walks the user through language, keyboard,
identity keys, a disk passphrase, persistence, network mode, an admin password, privacy toggles, and
appearance, then commits the collected configuration to the policy service and exits so the kernel brings
up the desktop. It is a direct compositor-surface runner, not an [app-skeleton](../writing-an-app.md)
window. The source is `userland/capsule_setup_wizard/`.

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

The wizard is the machine's first-run screen. On a fresh boot the orchestrator brings up the GUI core
(compositor, window manager, input router, desktop shell, policy), spawns the wizard, and holds the rest
of the desktop back until the wizard exits (`src/userspace/init/spawn_plan/orchestrator.rs:55`,
`orchestrator.rs:67`). While it runs it owns the whole screen: it registers one full-display surface with
the compositor, grabs the keyboard from the input router so nothing else can observe first-run entry, and
drives a ten-step state machine on key presses (`src/setup/mod.rs:44`, `src/server/runner.rs:14`).

There is no window and no app skeleton. `_start` initialises the heap, runs `setup::run` to discover
services and put a surface on screen, then hands the resulting `Context` to `server::runner::run`, which
never returns (`src/main.rs:16`). Each step paints a left rail with the step list and a right card with
that step's content; Enter advances, Escape goes back, and per-step keys make the choice. When the last
step commits, the wizard writes every choice to the policy service and calls `mk_exit(0)`, after which the
supervisor starts the desktop (`src/render/screens/review.rs:21`, `src/server/runner.rs:31`,
`src/userspace/init/supervisor/loop_impl.rs:36`).

## Identity

Everything the kernel and the service registry need to name and reach the wizard comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `setup-wizard` | `Capsule.mk:6` |
| Service handle | `app.setup_wizard` | `Capsule.mk:7`, `src/userspace/capsule_setup_wizard/spawn.rs:31` |
| Namespace | `systems.nonos.app.setup_wizard` | `Capsule.mk:12` |
| Service endpoint | `service:4794:app.setup_wizard` | `Capsule.mk:13`, `spawn.rs:32` |
| Reply endpoint | `reply:4795:endpoint.app.setup_wizard.reply` | `Capsule.mk:14`, `spawn.rs:33`, `spawn.rs:34` |
| Capability mask | `0x1819` | `Capsule.mk:15` |
| Binary name | `setup_wizard` | `Capsule.mk:10` |
| Kernel mirror | `src/userspace/capsule_setup_wizard` | `Capsule.mk:16` |

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
(`src/userspace/capsule_setup_wizard/spawn.rs:50`). It is the same graphics-client mask the terminal and
the other full-screen leaf renderers carry: it can create a surface, ask the display for its size, and
speak IPC, and nothing more. There is no `GraphicsSurfaceMap`, no `GraphicsPresent`, no `Network` bit (4),
no `FileSystem` bit (64), and no `Crypto`, hardware, driver, or DMA capability. The wizard's power over the
system does not come from this mask; it comes from being one of two names the policy service trusts to
write, which the security section below sets out.

The wizard is one of the two capsules the policy service allows to write configuration. The policy
set-handler gate hard-codes exactly two trusted setters, `app.settings` and `app.setup_wizard`, and
rejects a write from any other pid with `E_ACCES` (`userland/capsule_policy/src/server/handle_set.rs:23`,
`handle_set.rs:41`). It is also one of the three names the input router allows to take an exclusive
keyboard grab, alongside `app.boot_splash` and `app.input_probe`
(`userland/capsule_input_router/src/server/handlers/grab_request.rs:25`).

## User reference

The wizard is a ten-step state machine. `screens::draw` selects the screen for the current `ctx.step`
and `screens::on_key` routes the key to that screen's handler (`src/render/screens/mod.rs:15`,
`src/render/screens/mod.rs:31`). The left rail lists all ten steps with a `+` for done, `>` for current,
and `.` for pending (`src/render/chrome.rs:18`, `src/render/theme.rs:15`). Across every step, Enter
advances and Escape goes back one step (`src/server/step.rs:22`); list steps also accept `j`/`k` to move
the selection down/up and `1`..`9` to jump directly to an item (`src/server/step.rs:30`). Escape on the
first step and Enter past the last are both clamped, so the machine never underflows or overshoots
(`src/server/step.rs:14`).

Step 0, Language (`src/render/screens/language.rs`). A single-select list of twelve languages, English
through Hindi, from the shared label table (`userland/policy_proto/src/language_labels.rs:17`). `j`/`k`
and `1`..`9` choose, Enter advances (`language.rs:15`). The choice is stored as an index in
`ctx.lang_sel` (`src/state.rs:16`).

Step 1, Keyboard layout (`src/render/screens/keyboard.rs`). A single-select list of ten layouts, US
QWERTY through Chinese, from the shared label table
(`userland/policy_proto/src/keyboard_layout_labels.rs:17`). Same navigation; the index lands in
`ctx.kbd_sel` (`keyboard.rs:24`).

Step 2, Identity keys (`src/render/screens/keygen.rs`). A two-stage progress screen for an Ed25519 and an
ML-DSA-65 keypair. The footer reads `ENTER GENERATE/NEXT` (`keygen.rs:10`). The first Enter marks the
Ed25519 task done and advances the progress bar to 50 percent; the second marks the ML-DSA-65 task done at
100 percent and sets `ctx.keys_done`; a third Enter moves to the next step (`keygen.rs:23`). This is a UI
stage counter, not key material: the screen advances `ctx.keygen_stage` and flips `ctx.keys_done` but does
not itself generate or store any key (`keygen.rs:26`).

Step 3, Disk-encryption passphrase (`src/render/screens/passphrase.rs`). A masked text field for the
passphrase that protects the persistent store. Any printable byte `0x20`..`0x7E` appends to `ctx.pass_buf`
up to 64 bytes, Backspace deletes the last byte, Enter advances (`passphrase.rs:27`). The field renders the
entered length as asterisks with a caret and a strength bar that fills 12 percent per character up to 100,
so it is masked and does show a coarse strength meter
(`src/render/widgets/field.rs:6`, `field.rs:29`). The bytes are held in `ctx.pass_buf`/`ctx.pass_len` and
are not written to policy (see the review step).

Step 4, Persistence (`src/render/screens/persistence.rs`). A two-item list: `Amnesic (RAM only)` or
`Persistent encrypted store` (`persistence.rs:5`). `j`/`k` and `1`/`2` choose, Enter advances; the index
lands in `ctx.persist_sel` (`persistence.rs:15`). The choice is recorded but the wizard itself does not
configure or encrypt a store.

Step 5, Network mode (`src/render/screens/network.rs`). A three-item list: `Amnesic / offline`, `Direct
connection`, or `Bridged / obfuscated` (`network.rs:5`). Same navigation; the index lands in `ctx.net_sel`
(`network.rs:20`). At review this index is mapped to two policy booleans, not stored raw.

Step 6, Administration password (`src/render/screens/admin.rs`). A masked field, identical mechanics to
the passphrase: printable bytes append to `ctx.admin_buf` up to 64 bytes, Backspace deletes, Enter advances,
with the same asterisk mask and strength bar (`admin.rs:27`, `src/render/widgets/field.rs:6`). The entered
admin password is collected into `ctx.admin_buf`/`ctx.admin_len` but is never committed to policy by the
review step.

Step 7, Privacy and hardening (`src/render/screens/privacy.rs`). A three-item toggle list: `MAC address
randomization`, `Secure-wipe RAM on shutdown`, and `Telemetry` (`privacy.rs:5`). `j`/`k` move the focus,
Space toggles the focused item, Enter advances (`privacy.rs:25`). The three toggle bits live in the low
byte of `ctx.privacy` and the focus index in its high byte; the field defaults to `0b0000_0011`, so the
first two toggles start on (`src/state.rs:59`, `privacy.rs:8`).

Step 8, Appearance (`src/render/screens/appearance.rs`). A three-item wallpaper list, `Deep`, `Slate`,
`Night`, chosen with `j`/`k` or `1`..`3` into `ctx.wall_sel`; the `t` key cycles the theme index
`ctx.theme_sel` through three values; Enter advances (`appearance.rs:5`, `appearance.rs:15`). Note these
are the wizard's own three built-in wallpaper names, not the full policy wallpaper catalog.

Step 9, Review (`src/render/screens/review.rs`). A confirmation screen showing three status lines,
`Identity keys` (ticked when `ctx.keys_done`), `Passphrase set` (ticked when `ctx.pass_len > 0`), and
`Layout/wallpaper chosen` (always ticked) (`review.rs:13`). Enter runs `commit` then advances past the
last step, which ends the wizard; Escape goes back to appearance (`review.rs:40`). `commit` is the trusted
policy write and is detailed under Protocol and IPC.

Two selections in `Context` have no screen that edits them. `ctx.tz_off` stays at its `0` default because
no step changes it, yet it is still written to policy as `Timezone` (`src/state.rs:55`, `review.rs:28`).
`ctx.host_buf`/`ctx.host_len` likewise stay empty because no step edits a hostname, so the `Hostname`
write is guarded by `host_len > 0` and never fires in the current flow (`review.rs:35`). Both are wired
into the commit ahead of any UI that would set them.

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. Its top-level modules are `clients` (the compositor, input-router,
display-info, and policy IPC clients), `protocol` (the input-router request and delivery wire), `render`
(the panel, screens, widgets, and theme), `server` (the key loop and step machine), `setup` (service
discovery and surface bring-up), and `state` (the `Context`) (`src/main.rs:6`).

`Context` is the whole model: the surface base pointer, width, height, and stride; the three discovered
service ports; the current step; and one field per selection, the passphrase, admin, and hostname byte
buffers, and the keygen stage (`src/state.rs:1`). There is no other persistent state and no second thread.

The step machine is deliberately small. `step::apply` maps an `Outcome` of `Advance`, `Back`, or `Stay`
onto the step counter, clamping at `0` and at `DONE = 10` (`src/server/step.rs:1`, `step.rs:14`).
`default_key` turns Enter into `Advance` and Escape into `Back`; `list_nav` turns `k`/`j` into up/down and
a digit `1`..`9` into a direct index, returning `Stay` (`step.rs:22`, `step.rs:30`). Each screen's
`on_key` first tries its own keys, then falls back to `default_key`.

Lifecycle:

1. The orchestrator, under the `microkernel-setup-wizard` feature, first spawns the GUI core, then boots
   the wizard through the shared capsule-boot path and holds the desktop fleet until it exits
   (`src/userspace/init/spawn_plan/orchestrator.rs:55`, `orchestrator.rs:67`,
   `src/userspace/init/spawn_plan/boot.rs:20`). Boot verifies the embedded ELF, id cert, manifest, and ZK
   attestation, registers `app.setup_wizard` on port 4794, and logs `[SETUP-WIZARD] capsule spawned`
   (`src/userspace/capsule_setup_wizard/spawn.rs:37`, `src/userspace/init/capsule_boot/run.rs:29`).
2. `setup::run` discovers the compositor and input-router ports (required) and the policy port (optional),
   health-checks the compositor, and queries the display size; it maps a backing buffer, registers a
   surface descriptor, shares the surface handle, fills the backdrop, submits the surface to the
   compositor at overlay Z, and commits the first damage rectangle (`src/setup/mod.rs:18`,
   `src/setup/discover.rs:22`, `src/clients/display_info.rs:34`).
3. `server::runner::run` subscribes to key events and grabs the keyboard, draws the first screen, then
   loops on `mk_ipc_recv_from`: it parses each delivery, ignores anything that is not a key-down, feeds
   the code to `screens::on_key`, applies the outcome to the step, and redraws (`src/server/runner.rs:12`,
   `src/protocol.rs:27`).
4. Redraw paints the current screen into the shared surface and commits a full-window damage rectangle to
   the compositor (`src/server/runner.rs:39`, `src/render/mod.rs:22`).
5. When the step reaches `DONE`, the runner removes the scene from the compositor and calls `mk_exit(0)`;
   the review step has already committed the choices, and the supervisor then starts the desktop
   (`src/server/runner.rs:31`, `src/render/screens/review.rs:42`).

## Protocol and IPC

The wizard registers a service endpoint but exposes no application opcodes of its own; everything it does
is an outbound IPC call. The calls it makes:

Service discovery, through `mk_service_lookup` (`src/setup/discover.rs:7`): `compositor` and `input_router`
are required and a missing one aborts setup with an error; `policy` is optional and a miss leaves
`policy_port = 0`, in which case the review commit is skipped entirely (`discover.rs:18`, `discover.rs:30`,
`src/render/screens/review.rs:22`).

Compositor, service `compositor`, magic `0x4E43_4D50`, version 1
(`src/clients/compositor/constants.rs:19`):

```
  OP_HEALTHCHECK    0x0001   liveness probe before bring-up      constants.rs:21
  OP_SCENE_SUBMIT   0x0002   put the wizard surface on screen    constants.rs:23
  OP_DAMAGE_COMMIT  0x0003   flush a damage rectangle each frame constants.rs:20
  OP_SCENE_REMOVE   0x0007   pull the surface on exit            constants.rs:22
  OP_DISPLAY_INFO   0x0008   query width/height/stride           display_info.rs:9
```

The submit path passes the shared surface handle, position, size, and overlay Z; the display-info reply
carries a status word then width, height, and stride, and a zero in any of the three is rejected
(`src/setup/mod.rs:53`, `src/clients/display_info.rs:45`).

Input router, service `input_router`, request magic `0x4E49_5253` ("NIRS"), delivery magic `0x4E49_4E50`
("NINP") (`src/protocol.rs:3`, `src/protocol.rs:10`): `OP_SUBSCRIBE 0x0002` and `OP_GRAB_REQUEST 0x0003`,
both sent with a keys-only kind mask of `0b11` (`src/protocol.rs:7`, `src/clients/input_router.rs:7`,
`input_router.rs:32`). Key deliveries arrive as `mk_ipc_recv_from` frames the runner parses through
`parse_delivery` (`src/server/runner.rs:19`, `src/protocol.rs:27`). The grab is the one that carries the
security weight; the router only honours it because the wizard is a named trusted grabber.

Policy, service `policy`, resolved at setup and used only by the review commit
(`src/clients/policy.rs`). Each setter builds a policy header and one typed payload byte (or a string
body) and calls `mk_ipc_call`, treating a non-zero reply status as an error it discards
(`policy.rs:6`, `policy.rs:19`). The review `commit` writes, in order (`src/render/screens/review.rs:21`):

```
  set_u8   Field::Language 0x010B        ctx.lang_sel                    review.rs:26
  set_u8   Field::KeyboardLayout 0x0107  ctx.kbd_sel                     review.rs:27
  set_i8   Field::Timezone 0x0109        ctx.tz_off (always 0)           review.rs:28
  set_u8   Field::Wallpaper 0x0117       ctx.wall_sel                    review.rs:29
  set_u8   Field::Theme 0x0106           ctx.theme_sel                   review.rs:30
  set_bool Field::AnonymousMode 0x0104   net_sel == 0                    review.rs:31
  set_bool Field::WifiAutoconnect 0x0114 net_sel == 1                    review.rs:32
  set_bool Field::AutoWipe 0x0108        privacy bit 0b010               review.rs:33
  set_bool Field::NymEnabled 0x0105      privacy bit 0b001               review.rs:34
  set_str  Field::Hostname 0x0301        ctx.host_buf (only if len > 0)  review.rs:36
```

The field numbers are the shared `Field` enum (`userland/policy_proto/src/field.rs:19`). Note that the
network step's three-way choice collapses into two booleans, so `Bridged / obfuscated` (index 2) sets
neither `AnonymousMode` nor `WifiAutoconnect`. The passphrase, the admin password, and the persistence
choice are collected but have no `set_*` call in `commit`, so they never reach policy through this path.

## Security analysis

The wizard's capability mask is ordinary. `0x1819` grants CoreExec, IPC, Memory, GraphicsDisplayQuery, and
GraphicsSurfaceCreate and nothing else (`Capsule.mk:15`,
`src/userspace/capsule_setup_wizard/spawn.rs:50`). It has no network, filesystem, crypto, hardware,
driver, MMIO, or DMA capability. On the strength of its mask alone it can register one surface, ask the
display for its geometry, and speak IPC, exactly like the terminal.

The authority that matters is on the policy side, not the wizard's. The policy service keeps a two-name
allowlist of trusted setters, `app.settings` and `app.setup_wizard`, and its set dispatcher looks up the
sender's pid against those names before it will apply any write; a write from any other capsule is
answered `E_ACCES` and dropped (`userland/capsule_policy/src/server/handle_set.rs:23`, `handle_set.rs:36`,
`handle_set.rs:41`). So the wizard's commits take effect because policy trusts this name, not because the
wizard holds any special capability. That boundary is policy's to enforce: the wizard cannot grant itself
any authority through this channel, and it can only write the fields the policy handlers accept, each of
which type-checks and range-checks its payload on policy's side (`handle_set.rs:45`). The [settings
app](../settings/README.md) is the other trusted writer and is how these values are changed after first boot; the
[policy](../policy/README.md) service is the store both write through.

The keyboard grab is the wizard's other privileged interaction, and it is likewise a grant the input
router makes by name. The router's grab handler hard-codes three trusted grabbers, `app.boot_splash`,
`app.setup_wizard`, and `app.input_probe`, and refuses a grab from anyone else with `E_ACCES`
(`userland/capsule_input_router/src/server/handlers/grab_request.rs:25`, `grab_request.rs:32`). The wizard
grabs the keyboard so first-run entry, including the passphrase and admin password, cannot be observed by a
background subscriber; if the keyboard is already held the router returns `E_BUSY` instead
(`grab_request.rs:48`). Again the boundary lives in the [input router](../input-router/README.md), gated on the
wizard's name.

What the wizard cannot do is as important as what it can. It records intent, it does not enact secrets. The
identity-keys step advances a UI stage without generating or storing any key
(`src/render/screens/keygen.rs:26`); the passphrase and admin fields are masked and hold their bytes only
in the wizard's own memory, and neither is written to policy or used to key a store
(`src/render/screens/passphrase.rs`, `src/render/screens/admin.rs`, `src/render/screens/review.rs:21`); and
the persistence choice is collected without the wizard configuring or encrypting any store. The
security-sensitive parts of first-run, real key material and an encrypted persistent store, are therefore
deferred to other capsules; the wizard collects the configuration and commits the non-secret parts to
policy. There is also no abort-and-reboot from the final review: the only forward exit is to commit and
finish (`review.rs:40`), and Escape steps back through the flow rather than cancelling it.

## How to contribute

The source lives at `userland/capsule_setup_wizard/`. The screens are under `src/render/screens/` (one
file per step), the shared widgets under `src/render/widgets/`, the step machine and key loop under
`src/server/`, the IPC clients under `src/clients/`, and the collected state in `src/state.rs`.

To add a new step:

1. Write the screen module under `src/render/screens/`, one file for the step, exposing `pub fn
   draw(ctx: &Context)` that paints through `render::frame` plus a widget, and `pub fn on_key(ctx: &mut
   Context, code: u32) -> Outcome` that handles the step's keys and falls back to `default_key` for Enter
   and Escape (follow `src/render/screens/network.rs` for a list step or `src/render/screens/admin.rs` for
   a text field).
2. Add any new selection field to `Context` and its default in `Context::new` (`src/state.rs:1`).
3. Register the module and wire both the `draw` and the `on_key` match arms in
   `src/render/screens/mod.rs:15` and `mod.rs:31`, add the step label to `STEP_LABELS`
   (`src/render/theme.rs:15`), and raise `DONE` in `src/server/step.rs:1` so the machine runs the new step
   before review.
4. If the choice should persist, add its `Field` to the shared enum
   (`userland/policy_proto/src/field.rs:19`) and a matching `set_*` call to the review `commit`
   (`src/render/screens/review.rs:21`); the write only lands because the wizard is a trusted policy setter.

To build and sign the capsule, use the generated per-slug make targets. The `setup-wizard` slug comes from
`Capsule.mk:6`, and `Capsule.mk:18` includes `nonos-mk/capsule.mk`, which instantiates the per-slug rules
(`nonos-mk/capsule.mk:158`):

```
  make nonos-mk-setup-wizard             build the capsule ELF
  make nonos-mk-setup-wizard-sign        produce the id cert, manifest, and attestation trailer
  make nonos-mk-setup-wizard-verify      verify the signed artifacts against the trust anchor
  make nonos-mk-check-setup-wizard-keys  check the per-capsule signing keys exist
```

For a bootable first-run image, `make nonos-mk-setup-wizard-prod` builds the full desktop GUI kernel under
the `microkernel-setup-wizard` feature with the wizard in the fleet, and `make nonos-mk-setup-wizard-esp`
packages that into an ESP; `make nonos-mk-run-wizard` boots the first-run wizard in QEMU under OVMF
(`Makefile:1099`, `Makefile:1151`, `Makefile:1298`). There is also a `nonos-mk-setup-wizard-inject-prod`
profile that layers the input-probe inject feature on top for on-hardware input bring-up
(`Makefile:1121`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every client call returns a `Result` the wizard discards or acts on, never a
panic); modular files, one unit per file, with `mod.rs` used only for module declarations and re-exports
(`src/render/screens/mod.rs`, `src/clients/compositor/mod.rs`); and the AGPL header at the top of every
source file, matching the header already on the modules in this tree
(`src/setup/fill.rs:1`).

## Debugging

The first thing to confirm is that the wizard ran. On a boot under the `microkernel-setup-wizard` feature
the kernel prints `[SETUP-WIZARD] capsule spawned` from the shared capsule-boot path (prefix
`SETUP-WIZARD`, message `capsule spawned`) (`src/userspace/init/spawn_plan/orchestrator.rs:59`,
`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line means the
capsule never started, usually a signature, manifest, or capability failure; the error path prints a
prefixed `[ERROR]` line built from the spawn error instead
(`src/userspace/init/capsule_boot/run.rs:32`, `src/userspace/init/capsule_boot/error.rs:21`).

Failure modes and where to look:

- The screen stays black or the wizard exits immediately. Bring-up aborts with a specific error string if
  a required service is missing or the display query fails: the compositor or input-router lookup, the
  compositor health check, the display-info query, the backing mmap, the surface register or share, or the
  scene submit, each returns its own message and `_start` exits with code 2
  (`src/setup/mod.rs:18`, `src/main.rs:22`).
- The wizard is on screen but does not respond to Enter or Escape. That is the keyboard grab failing, not
  the state machine wedging. The router refuses a grab with `E_ACCES` if the sender is not resolved as a
  trusted grabber, or `E_BUSY` if the keyboard is already held, and the wizard then receives no key
  deliveries (`userland/capsule_input_router/src/server/handlers/grab_request.rs:32`, `grab_request.rs:48`,
  `src/server/runner.rs:14`). The subscribe and grab return values are discarded, so the symptom is silence
  rather than an error line.
- Settings do not stick after first boot. The commit is skipped whole if the policy service was absent at
  discovery, because `policy_port` is then `0` (`src/render/screens/review.rs:22`,
  `src/setup/discover.rs:30`). If policy is present but a specific value is wrong, check the mapping in
  `commit`: the network step collapses to two booleans, and the timezone and hostname fields have no UI in
  the current flow, so `Timezone` is always written `0` and `Hostname` is never written
  (`src/render/screens/review.rs:28`, `review.rs:35`). A rejected individual write returns a non-zero
  policy status the wizard discards, so a single bad field fails silently rather than aborting the commit
  (`src/clients/policy.rs:12`).
- The wizard finishes but the desktop does not appear. The wizard exiting is the trigger for the desktop
  fleet: the supervisor only starts the desktop once the wizard's shared state is no longer alive
  (`src/userspace/init/supervisor/loop_impl.rs:36`, `src/userspace/init/spawn_plan/orchestrator.rs:67`). A
  wizard that completes disappears cleanly via `mk_exit(0)` (`src/server/runner.rs:33`); one that hangs
  before review is holding the desktop back.

## Source map

```
  src/main.rs                              _start -> setup::run -> server::runner::run
  src/state.rs                             Context: step, every selection, pass/admin/host buffers
  src/setup/mod.rs                         discover services, register + submit the surface
  src/setup/discover.rs                    compositor/input_router required, policy optional
  src/setup/fill.rs                        backdrop fill of the backing buffer
  src/server/runner.rs                     the key-driven loop, subscribe, grab, redraw, mk_exit
  src/server/step.rs                       Outcome/apply, default_key, list_nav, DONE
  src/render/mod.rs                        frame: backdrop, card, panel, title, footer
  src/render/chrome.rs                     the left rail: wordmark and step list
  src/render/screens/                      the ten steps (language .. review)
  src/render/screens/review.rs             the policy commit (the trusted write)
  src/render/widgets/                      list, toggles, masked field, progress widgets
  src/render/theme.rs                      colors and STEP_LABELS
  src/clients/compositor/                  compositor client (health, submit, damage, remove)
  src/clients/display_info.rs              display geometry query
  src/clients/input_router.rs              subscribe and grab_keyboard
  src/clients/policy.rs                    set_bool/set_u8/set_i8/set_str
  src/protocol.rs                          input-router request and key-delivery wire
  Capsule.mk                               slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_setup_wizard/      the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/orchestrator.rs   the first-run spawn ordering
  userland/capsule_policy/src/server/handle_set.rs   the two-name policy write gate
  userland/capsule_input_router/src/server/handlers/grab_request.rs   the three-name grab gate
  userland/policy_proto/src/field.rs       the shared policy Field enum
  nonos-mk/capsule.mk                      the generated nonos-mk-setup-wizard[-sign|-verify] targets
```

Every reference above is verified against those trees.

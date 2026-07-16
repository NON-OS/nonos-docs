# capsule_boot_splash (full reference)

`capsule_boot_splash` is the first thing a NONOS user sees. The loader hands off to the kernel, the
kernel brings up the display core, and before the desktop fleet finishes spawning this capsule paints a
fullscreen splash that carries the boot-chain attestation status. It registers no service of its own; it
is a pure compositor client that draws a splash, reads the kernel's attestation, holds the screen while
the desktop comes up, and exits to hand off. The source is `userland/capsule_boot_splash/`; the short
version is the [desktop overview](desktop.md).

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Behavior reference](#behavior-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The splash exists to fill the gap between the loader handing off and the desktop being ready to paint.
The display core (input router and compositor) is brought up before the driver and network fleets
specifically so the splash can appear immediately and hold while the rest of the capsules spawn behind it
(`src/userspace/init/spawn_plan/desktop_fleet.rs:27`, `:32`). `_start` initializes the heap and runs a
single linear sequence, with no long-lived server loop (`src/main.rs:31`, `src/main.rs:38`):

```
  run():
      comp = wait_compositor()                      // lookup + healthcheck, up to 256 tries
      (w, h, stride) = display::query(comp)          // NCMP display-info
      (base, handle) = surface::setup(w, h, stride)  // mmap + register + share one surface
      badge = mk_attest_status(&att) == 0 ? Some(att.zk_verified == 1) : None
      paint::splash(base, w, h, stride, badge)
      scene::submit(comp, handle); scene::damage(comp)
      grabbed_interact(...)                          // grab keys, spin, wait for desktop_shell, D toggles detail
      scene::remove(comp); mk_surface_release(handle)
```

The distinctive job is the attestation badge. The splash reads the kernel's attestation status and paints
a boot-chain panel with an `ATTESTED`, `UNVERIFIED`, or `verifying` verdict, and `D` opens a detail view
with the kernel hash and the ZK program hash (`src/main.rs:52`, `src/paint.rs:48`, `src/detail.rs:33`).
The honest framing is the same one the [attest](attest.md) page carries: the splash **displays** the
kernel's attestation, it does not verify a proof itself. The real check lives in the kernel and the
bootloader, documented under the [proof system](../../subsystems/proof-system/README.md).

## Identity

Everything the kernel and the service registry need to name the splash comes from its `Capsule.mk` and
its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `boot-splash` | `Capsule.mk:8` |
| Service handle | `app.boot_splash` | `Capsule.mk:9`, `src/userspace/capsule_boot_splash/spawn.rs:31` |
| Namespace | `systems.nonos.app.boot_splash` | `Capsule.mk:14` |
| Service endpoint | `service:4796:app.boot_splash` | `Capsule.mk:15`, `spawn.rs:32` |
| Reply endpoint | `reply:4797:endpoint.app.boot_splash.reply` | `Capsule.mk:16`, `spawn.rs:33`, `:34` |
| Capability mask | `0x1819` | `Capsule.mk:17` |
| Binary name | `boot_splash` | `Capsule.mk:12` |
| Kernel mirror | `src/userspace/capsule_boot_splash` | `Capsule.mk:18` |

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
(`src/userspace/capsule_boot_splash/spawn.rs:50`). This is the same leaf-renderer mask the
[wallpaper](wallpaper.md), [login](login.md), and [setup wizard](setup-wizard.md) capsules carry: it can
query the display, register one surface, and speak IPC, and nothing else. There is no `GraphicsSurfaceMap`
bit (8192), no `GraphicsPresent` bit (16384), no `Network`, `FileSystem`, `Crypto`, hardware, driver, or
DMA bit.

Note the one identity quirk that the [desktop overview](desktop.md) flags: the `Capsule.mk` reserves
`service:4796:app.boot_splash` and a reply port on 4797, and the kernel registers the service and its
reply inbox at spawn (`spawn.rs:31`), but the capsule code never binds or receives on 4796. It is a pure
client, so the reserved endpoint exists in the record but is never the surface anyone looks it up on. The
verification for that claim is direct: nothing in `src/` calls a bind or a receive keyed on 4796, and the
only receive the loop makes is on port 0 (`src/main.rs:112`).

## Behavior reference

### What it draws

The frame is composed over a radial vignette background: a dark core that lerps out to black, computed per
pixel, filling the whole surface (`src/vignette.rs:20`, `:24`, and the `lerp` at `:38`). On top of that
the splash paints (`src/paint.rs:30`):

- The wordmark `NØNOS`, drawn at scale 5 with a one-pixel dark drop shadow behind the accent glyphs,
  centered horizontally (`src/paint.rs:35`, `:39`, `:40`).
- The subtitle `zero-state attestation boot`, centered under the wordmark in a dim color
  (`src/paint.rs:41`).
- The attestation panel, a bordered box titled `boot-chain attestation` with three fixed status lines
  (`[+] bootloader  ed25519 verified`, `[+] kernel      blake3 verified`, `[#] capsules    zk attested`)
  and one live verdict line drawn from the kernel status (`src/paint.rs:48`). The panel chrome (border,
  title, and the rule under the title) is drawn by `chrome::panel` (`src/chrome.rs:33`).
- A status line low on the screen with a spinner glyph and the label `initializing zero-state`
  (`src/paint.rs:64`).

The verdict line is the security-relevant pixel. It reads `ATTESTED` in green when the kernel reported
`zk_verified == 1`, `UNVERIFIED` in amber when it reported `0`, and `verifying` in dim when the status
read failed (`badge == None`) (`src/paint.rs:56`, set in `src/main.rs:52`).

### The animation and its timing

The spinner is a time-based cycle, not a progress metric. The loop computes a frame number from elapsed
milliseconds (`el / 150`) and, whenever that number changes, redraws only the status band with the next
spinner glyph from `|/-\` and a blinking cursor (`src/main.rs:127`, `src/paint.rs:64`, `:68`, `:69`). It
repaints just the band, not the whole frame, using `vignette::fill_band` to reset the strip before
redrawing (`src/paint.rs:66`, `src/vignette.rs:24`). A spinning splash therefore means the capsule is
alive and cycling; it is not evidence of forward progress in the boot.

### The detail view

Pressing `D` (upper or lower case) toggles a detail view (`src/main.rs:116`). It repaints the surface with
a `boot-chain attestation` panel showing the kernel BLAKE3 hash and the ZK program hash rendered as hex,
plus two colored flags, `secure_boot` and `attested`, green when set and amber when not
(`src/detail.rs:33`, `:41` through `:47`). The footer reads `press any key to return`
(`src/detail.rs:48`). Any key press with the detail already open toggles it back to the splash
(`src/main.rs:116`, `:120`). While the detail view is open the handoff is suppressed, so a held detail
view will not let the splash exit out from under the reader (`src/main.rs:108`, guarded by `!show_detail`).

### When and how it exits

The loop watches for the `desktop_shell` service to register. Once it appears, the splash records the
time, waits a one-second settle so the desktop paints behind the overlay, and then hands off
(`src/main.rs:103`, `:106`, `SETTLE_MS` at `src/main.rs:25`). There are two other exits: a hard cap of
thirty seconds of dwell (`MAX_DWELL_MS`, `src/main.rs:26`, `:108`), and an iteration ceiling of eight
million loop passes (`MAX_ITERS`, `src/main.rs:27`, `:108`), either of which breaks the loop so the splash
can never hang. On exit it removes its scene from the compositor and releases the surface
(`src/main.rs:60`, `:61`). If the compositor lookup or healthcheck never succeeds the splash never paints
and returns rather than blocking (`src/main.rs:39`, `wait_compositor` at `:76`).

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. The nine modules are `chrome`, `detail`, `display`, `input`, `paint`,
`proto`, `scene`, `surface`, and `vignette` (`src/main.rs:6` through `:14`). There is no app-skeleton and
no `App` trait here; unlike the [terminal](terminal-full.md), the splash drives its own surface and its
own loop directly, because it is an overlay that predates the window manager rather than a normal window.

- `proto` is the NCMP client core: the `0x4E43_4D50` magic, the 20-byte header builder, the
  `call_status` request-reply helper, the service `lookup`, and the compositor healthcheck
  (`src/proto.rs:22`, `:27`, `:39`, `:53`, `:63`).
- `display` issues the display-info query and validates the returned width, height, and stride
  (`src/display.rs:30`).
- `surface` mmaps an anonymous private buffer sized `stride * height`, registers it as an ARGB8888
  surface, and shares it to get a compositor handle (`src/surface.rs:24`, `:47`, `:51`).
- `scene` submits the surface to the compositor at a fixed overlay Z, commits damage, and removes the
  scene (`src/scene.rs:27`, `:39`, `:48`; `OVERLAY_Z = 4_000_000` at `:25`).
- `input` speaks the input-router `0x4E49_5253` request protocol (subscribe, grab, release) and parses
  the `0x4E49_4E50` key-delivery frames (`src/input.rs:52`, `:56`, `:60`, `:64`).
- `paint`, `detail`, `chrome`, and `vignette` are the renderer, drawing through the shared
  `nonos_toolkit` font atlas (`src/paint.rs:17`, `src/chrome.rs:17`).

Lifecycle:

1. The kernel spawns the capsule at boot through the desktop-fleet plan, which brings up the display core
   early and calls the idempotent `spawn_boot_splash` guarded by an `is_alive` check
   (`src/userspace/init/spawn_plan/desktop_fleet.rs:32`, `:43`, `:45`). The spawn verifies the embedded
   ELF, id cert, manifest, and attestation trailer and registers `app.boot_splash` on port 4796
   (`src/userspace/capsule_boot_splash/spawn.rs:37`, `:57`). On success the boot path logs
   `[BOOT-SPLASH] capsule spawned` (`src/userspace/init/spawn_plan/desktop_fleet.rs:49`,
   `src/userspace/init/capsule_boot/run.rs:29`).
2. `run` waits for the compositor, queries the display, sets up the surface, reads the attestation, paints
   the first frame, submits and damages the scene, and enters the interaction loop
   (`src/main.rs:38` through `:62`).
3. `grabbed_interact` looks up the input router; if it is present the splash subscribes to keys and takes
   an exclusive key grab for the duration, then releases it on the way out
   (`src/main.rs:65`, `:70`, `:71`, `:73`). If the router is absent it interacts without a grab
   (`src/main.rs:68`).
4. The loop animates the spinner, processes any key delivery (the `D` toggle), and watches for
   `desktop_shell` (`src/main.rs:88` through `:137`).
5. On exit the scene is removed and the surface released, and the splash returns, freeing the screen for
   the desktop (`src/main.rs:60`, `:61`).

## Protocol and IPC

The splash exposes no application opcodes. Everything it does that leaves the capsule is an outbound IPC
call. It talks to exactly two services.

Compositor, service `compositor`, NCMP magic `0x4E43_4D50`, version 1, 20-byte header
(`src/proto.rs:22`, `:23`, `:24`):

```
  OP_HEALTHCHECK    0x0001    liveness probe during wait_compositor   proto.rs:25
  OP_SCENE_SUBMIT   0x0002    attach the shared surface at overlay Z  scene.rs:19
  OP_DAMAGE_COMMIT  0x0003    commit a damage rectangle               scene.rs:20
  OP_SCENE_REMOVE   0x0007    detach the scene on exit                scene.rs:21
  OP_DISPLAY_INFO   0x0008    query width, height, stride             display.rs:23
```

The submit places the surface at `OVERLAY_Z = 4_000_000` so the splash sits above ordinary windows while
it holds the screen (`src/scene.rs:25`, `:34`). The display query rejects a zero width, height, or stride
(`src/display.rs:43`). Every compositor call goes through `call_status`, which reads back a 24-byte
header-plus-status reply and treats a nonzero status as an error (`src/proto.rs:39`).

Input router, service `input_router`, request magic `0x4E49_5253`, delivery magic `0x4E49_4E50`
(`src/input.rs:19`, `:26`):

```
  OP_SUBSCRIBE      0x0002    subscribe to the key kinds (mask 0b11)  input.rs:22, :52
  OP_GRAB_REQUEST   0x0003    take an exclusive key grab              input.rs:23, :56
  OP_GRAB_RELEASE   0x0004    release the grab on exit                input.rs:24, :60
```

Key events are not received on the reserved service port. The subscribe-and-grab arrangement causes the
router to deliver key frames to the capsule's inbox, and the loop reads them with `mk_ipc_recv_from(0,
...)` on port 0 with a 50 ms timeout, then parses the 40-byte delivery frame for the key kind and code
(`src/main.rs:112`, `src/input.rs:27`, `:64`). The kernel status read is a plain syscall, not IPC:
`mk_attest_status` fills an `AttestStatus` with `zk_verified`, `kernel_sig_ok`, `secure_boot`,
`zk_attestation_ok`, and the two 32-byte hashes (`userland/libc/src/attest.rs:21`, called at
`src/main.rs:52`).

## Security analysis

The splash is deliberately one of the least privileged capsules in the tree. Its mask `0x1819` grants
CoreExec, IPC, Memory, GraphicsDisplayQuery, and GraphicsSurfaceCreate and nothing else
(`Capsule.mk:17`, `src/userspace/capsule_boot_splash/spawn.rs:50`). Beyond mmapping one surface, drawing
into it, talking to the compositor and the input router, and reading one status word, it cannot reach
anything. It holds no crypto, filesystem, network, hardware, driver, MMIO, or DMA capability, and it holds
no `GraphicsPresent`, so it never touches a framebuffer directly; the compositor owns presentation.

- **It displays, it does not verify.** This is the point worth restating. The badge is
  `mk_attest_status`'s `zk_verified` field, read out of the kernel and painted verbatim
  (`src/main.rs:52`, `src/paint.rs:56`). The splash has no `Crypto` capability and runs no verifier. The
  trust in that green `ATTESTED` line is the kernel's and the bootloader's attestation, not the splash's.
  A compromised splash could paint a green badge on an unverified system, but it could not make the system
  actually verified, and it could not weaken a real verification, because it is strictly downstream of the
  measurement. The [attest](attest.md) page and the [proof system](../../subsystems/proof-system/README.md)
  carry the real check.
- **The key grab is a router grant, not a splash power.** The splash takes an exclusive key grab so it can
  read the `D` toggle without other windows stealing focus (`src/main.rs:71`). That grab is only honored
  because the input router trusts the caller by name: its grabber allowlist is exactly three names,
  `app.boot_splash`, `app.setup_wizard`, and `app.input_probe`, checked against the live pid before any
  grab is granted (`userland/capsule_input_router/src/server/handlers/grab_request.rs:25`, `:27`, `:32`).
  A capsule not on that list gets `E_ACCES`. The authority lives in the router, gated by identity; the
  splash merely qualifies, and it releases the grab and exits when the shell comes up
  (`src/main.rs:73`).
- **Isolation is the kernel's.** The surface is a private anonymous mapping the splash registers and later
  releases (`src/surface.rs:26`, `:47`, `src/main.rs:61`); it holds no persistent state and writes no
  files. Separation from every other capsule is enforced by the kernel: the splash is a CPL 3 user binary
  verified and enrolled at spawn like every other capsule, and it only ever speaks IPC and its own
  surface.

## How to contribute

The source lives at `userland/capsule_boot_splash/`. To change what the splash looks like, the modules are
small and single-purpose:

- The layout and text of the splash (wordmark, subtitle, attestation panel, verdict wording, status line)
  is `src/paint.rs`. The verdict strings and their colors are the `match attested` in `attest_panel`
  (`src/paint.rs:56`).
- The detail view (which hashes and flags it shows) is `src/detail.rs`.
- The background gradient is `src/vignette.rs`; the panel border and title chrome is `src/chrome.rs`.
- The animation cadence is the `el / 150` frame divisor and `SPIN` in `src/main.rs:128` and
  `src/paint.rs:28`; the dwell and settle timing are the constants at the top of `src/main.rs:25`.

If you change the protocol wire (a new compositor or router op), edit `src/scene.rs`, `src/display.rs`, or
`src/input.rs` and keep the header builder in `src/proto.rs` as the single source of the NCMP header.

To build and sign the capsule, use the generated per-slug make targets. The rules are produced from the
slug `boot-splash` by `nonos-mk/capsule.mk` (the `.PHONY` line at `nonos-mk/capsule.mk:158` and the
recipes at `:184`, `:261`, `:263`), included through `userland/capsule_boot_splash/Capsule.mk:20`:

```
  make nonos-mk-boot-splash              build the capsule ELF
  make nonos-mk-boot-splash-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-boot-splash-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-boot-splash-keys   check the per-capsule signing keys exist
```

There is no `boot-splash`-specific prod image target. The splash ships as a component of the desktop
images: `make nonos-mk-desktop-gui-prod` and `make nonos-mk-full-gui-prod` both pull in
`$(boot-splash_ARTIFACTS)` (`Makefile:640`, `Makefile:1082`, `Makefile:1112`, `Makefile:1134`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every fallible step returns a `Result` or an `Option` and the run path turns a
failure into a numeric exit code, never a panic; see the `match` arms in `src/main.rs:38` through `:57`);
modular files, one unit per file; and the AGPL header at the top of every source file, matching the header
already on every module here (`src/paint.rs:1`).

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[BOOT-SPLASH] capsule spawned` from the boot path (tag `BOOT-SPLASH`, message `capsule spawned`)
(`src/userspace/init/spawn_plan/desktop_fleet.rs:49`, `src/userspace/init/capsule_boot/run.rs:29`,
`src/sys/boot_log/output.rs:33`). A signature, manifest, or capability failure prints an `[ERROR]` line
instead (`src/userspace/init/capsule_boot/run.rs:32`, `src/sys/boot_log/output.rs:49`). The lower-level
spawn also emits `[SPAWN] name=app.boot_splash pid=0x... caps=0x1819 entry=0x...`; the `caps=0x1819`
confirms the admitted mask (`src/kernel_core/process_spawn/capsule_spawn/runner/install/spawn_log.rs:17`).

Failure modes and where to look:

- The splash never appears. The capsule waits on `lookup("compositor")` plus a healthcheck up to 256
  times, and if the compositor never registers it returns exit code 2 without painting rather than hanging
  (`src/main.rs:39`, `wait_compositor` at `:76`, `READY_ATTEMPTS` at `:28`). A missing splash with the
  compositor absent is the display core failing to come up, not the splash. Exit codes 4, 5, and 6 isolate
  the next steps: 4 is a bad display query, 5 is a surface setup failure, 6 is a scene-submit rejection
  (`src/main.rs:44`, `:49`, `:56`).
- The splash stays on screen and never hands off. Handoff is gated on `desktop_shell` registering plus a
  one-second settle, or the thirty-second dwell cap (`src/main.rs:103`, `:106`, `:108`). A splash that
  clears quickly to a live desktop means `desktop_shell` registered on schedule; a splash that hangs on
  screen for the full dwell before clearing means the shell never registered and the hard cap fired. Note
  that an open detail view intentionally suppresses handoff (`src/main.rs:108`), so a splash that will not
  leave may simply have the `D` detail open.
- The spinner is turning but nothing else changes. That is expected. The spinner is a time-based animation
  driven off elapsed milliseconds, not a progress bar, so motion only proves the loop is running, not that
  the boot is advancing (`src/main.rs:128`, `src/paint.rs:64`).
- The badge reads `verifying` and never resolves. The verdict is `None` only when `mk_attest_status`
  returned nonzero, which leaves the badge dim (`src/main.rs:52`, `src/paint.rs:60`). An `UNVERIFIED`
  amber badge, by contrast, is a real read that reported `zk_verified == 0`, and that is a kernel or
  bootloader attestation question, not a splash bug.
- The `D` key does nothing. Key delivery depends on the input-router grab. If the router is absent the
  splash runs without a grab and never receives keys (`src/main.rs:68`); if the grab is denied the caller
  is not on the router's three-name allowlist (`grab_request.rs:25`, `:32`). Keys arrive on port 0, not
  the reserved service port (`src/main.rs:112`).

## Source map

```
  userland/capsule_boot_splash/src/main.rs      _start, run, the interaction loop, handoff, D toggle
  userland/capsule_boot_splash/src/proto.rs     NCMP header, call_status, lookup, healthcheck
  userland/capsule_boot_splash/src/display.rs   OP_DISPLAY_INFO query
  userland/capsule_boot_splash/src/surface.rs   mmap + register + share the single splash surface
  userland/capsule_boot_splash/src/scene.rs     scene submit / damage / remove at overlay Z
  userland/capsule_boot_splash/src/input.rs     router subscribe / grab / release, key-frame parse
  userland/capsule_boot_splash/src/paint.rs     the splash frame and the attestation verdict
  userland/capsule_boot_splash/src/detail.rs    the D-key detail view (kernel + ZK hashes, flags)
  userland/capsule_boot_splash/src/chrome.rs    panel border and title chrome
  userland/capsule_boot_splash/src/vignette.rs  the radial background fill and band redraw
  userland/capsule_boot_splash/Capsule.mk       slug, handle, reserved endpoint, capability mask
  userland/libc/src/attest.rs                   AttestStatus and mk_attest_status
  src/userspace/capsule_boot_splash/spawn.rs    the kernel-side verified spawn and requested caps
  src/userspace/init/spawn_plan/desktop_fleet.rs   the early-display fleet entry
  userland/capsule_input_router/src/server/handlers/grab_request.rs   the three-name grab allowlist
  nonos-mk/capsule.mk                           the generated nonos-mk-boot-splash[-sign|-verify] targets
```

Every reference above is verified against those trees.
</content>
</invoke>

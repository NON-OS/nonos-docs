# capsule_about (full reference)

`capsule_about` is the "About NONOS" application: a small GUI window that shows the product identity, the
running build, the capsule's own capability mask, the primary display, the uptime clock, and the full
AGPL-3 license text. It is a read-only introspection app. Everything it displays is either baked into the
binary at build time or read live from the kernel through two syscalls, and it never writes anything back.

It is an [app-skeleton](../writing-an-app.md) GUI app. The kernel spawns it under service handle
`app.about` on service port 4710 with a reply port on 4711, and its capability mask is `0x1819`
(`userland/capsule_about/Capsule.mk:11`). The source is `userland/capsule_about/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [User reference](#user-reference)
- [Keybindings and interaction](#keybindings-and-interaction)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The about app is an ordinary NONOS GUI application. Its entry point hands its `App` implementation to the
skeleton's `run`, so the runtime owns the surface, the window, the input subscription, and the paint
loop, and the app supplies three things: a manifest for a normal window, an `on_event` that turns
keystrokes and tab clicks into section changes and scrolling, and a `paint` that draws the frame
(`userland/capsule_about/src/main.rs:28`, `src/about/app.rs:34`).

The window is a tabbed panel. It carries exactly five sections, always in the same order: Identity,
Authority, Display, Uptime, and License (`src/about/section.rs:26`). Tab and Shift-Tab cycle through them,
a pointer click on the tab strip jumps to one directly, and the arrow, page, home, and end keys scroll the
body of a section that is taller than the window. There is no text entry, no command line, and no state
that survives a restart. The only outbound work the app does is two syscalls: it asks the kernel for the
primary display size and for the wall-clock milliseconds, and it renders the answers
(`src/about/data/display.rs:22`, `src/about/data/uptime.rs:20`).

## Identity

Everything the kernel and the service registry need to name and reach the about app comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `about` | `Capsule.mk:1` |
| Service handle | `app.about` | `Capsule.mk:2`, `src/userspace/capsule_about/spawn.rs:30` |
| Namespace | `systems.nonos.app.about` | `Capsule.mk:7` |
| Service endpoint | `service:4710:app.about` | `Capsule.mk:8`, `spawn.rs:31` |
| Reply endpoint | `reply:4711:endpoint.app.about.reply` | `Capsule.mk:9`, `spawn.rs:32` |
| Capability mask | `0x1819` | `Capsule.mk:11` |
| Binary name | `about` | `Capsule.mk:5` |
| Kernel mirror | `src/userspace/capsule_about` | `Capsule.mk:12` |

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
(`src/userspace/capsule_about/spawn.rs:49`). There is no `Network` bit (4), no `FileSystem` bit (64), and
no hardware, driver, MMIO, IRQ, DMA, or PIO capability in the mask. The app can create a surface, ask the
display for its size, and speak IPC, and that is all. The Authority section renders this exact mask from
the app's own copy of the capability table and marks every other capability as denied
(`src/about/data/caps.rs:47`, `src/about/section_render/authority.rs:52`).

## User reference

The window is a header, a tab strip, a scrollable body, a scrollbar, and a status bar. The header shows
the product name, the tagline, and a `n / 5` breadcrumb of the current section; the status bar shows a
fixed hint line; the body shows whichever section is selected (`src/about/paint/frame.rs:27`). Every
value below is what the user actually sees, enumerated from source.

### Header and status bar

The header fills the top of the window and always shows the same two labels plus a section counter
(`src/about/paint/header.rs:28`):

| Field | Value | Source |
|---|---|---|
| Product name | `NONOS` | `header.rs:30`, `data/product.rs:17` |
| Tagline | `Capability-based RAM-resident microkernel` | `header.rs:31`, `data/product.rs:18` |
| Breadcrumb | `<index+1> / 5`, right-aligned | `header.rs:34`, `section.rs:26` |

The status bar at the bottom is a single fixed hint string:
`Tab/Shift-Tab cycle sections   Up/Down scroll line   PgUp/PgDn scroll page   Esc close`
(`src/about/paint/status_bar.rs:21`).

### Identity section

Nine label/value rows. The version and commit are stamped at build time; the rest are compile-time
constants (`src/about/section_render/identity.rs:25`):

| Row | Value | Source |
|---|---|---|
| Product | `NONOS` | `identity.rs:26`, `data/product.rs:17` |
| Tagline | `Capability-based RAM-resident microkernel` | `identity.rs:27`, `data/product.rs:18` |
| Homepage | `https://nonos.systems` | `identity.rs:28`, `data/product.rs:20` |
| Copyright | `(c) 2026 NONOS Contributors` | `identity.rs:29`, `data/product.rs:19` |
| Version | the crate's `CARGO_PKG_VERSION` (currently `0.1.0`) | `identity.rs:30`, `data/build.rs:17`, `Cargo.toml:11` |
| Commit | the 12-char git SHA baked at build time | `identity.rs:31`, `data/build.rs:18`, `build.rs:22` |
| Toolchain | `nightly-2026-01-16` | `identity.rs:32`, `data/build.rs:19` |
| Architecture | `x86_64`, `aarch64`, or `riscv64` per the build target | `identity.rs:33`, `data/build.rs:21` |
| ABI | `Mk` | `identity.rs:34`, `data/abi.rs:17` |

The commit string is resolved by the build script: it prefers `NONOS_BUILD_SHA`, then `GITHUB_SHA`, then
`git rev-parse --short=12 HEAD`, and falls back to `unknown` if none of those produce a value
(`build.rs:30`).

### Authority section

This is the capsule's own capability report. It opens with six fixed rows describing the trust chain, then
lists every granted capability with its role, a `Denied:` marker, and every capability the mask does not
grant (`src/about/section_render/authority.rs:31`):

| Row | Value | Source |
|---|---|---|
| Chain | `trust-anchor -> publisher -> capsule` | `authority.rs:38`, `data/trust.rs:20` |
| Scheme | `Ed25519 + ML-DSA-65 (hybrid)` | `authority.rs:39`, `data/trust.rs:17` |
| Manifest | `capsule_manifest v3` | `authority.rs:40`, `data/trust.rs:18` |
| Cert | `NONOS-ID cert hybrid` | `authority.rs:41`, `data/trust.rs:19` |
| Status | `reached _start, which means capsule_spawn::spawn_verified accepted the cert + manifest` | `authority.rs:42`, `data/trust.rs:21` |
| Cap mask | the decimal value of the mask (`6169` for `0x1819`) | `authority.rs:43`, `data/caps.rs:47` |

After the fixed rows it prints the granted capabilities, each as `name` plus a one-line role
(`src/about/data/caps.rs:23`, filtered by `is_granted` at `authority.rs:52`):

```
  CoreExec               run user code
  IPC                    toolkit calls + event recv
  Memory                 mmap the paint buffer
  GraphicsDisplayQuery   learn display dimensions
  GraphicsSurfaceCreate  register the paint surface
```

Then a `Denied:` header, and the name of every capability the mask does not hold, so the user can see the
full table of what the app is not allowed to do: IO, Network, Crypto, FileSystem, Hardware, Debug, Admin,
RegisterService, GraphicsSurfaceMap, GraphicsPresent, DeviceEnum, Driver, Mmio, Irq, Dma, and Pio
(`src/about/section_render/authority.rs:59`, `data/caps.rs:23`). The row count grows with the table, so
adding a capability descriptor automatically lengthens the section (`authority.rs:25`).

Note that `GraphicsPresent` and `GraphicsSurfaceMap` land in the Denied list even though the app clearly
paints to the screen. The app registers a surface and the runtime and compositor do the mapping and the
scanout flush on its behalf; the about app itself never holds the present or map capability
(`data/caps.rs:47`).

### Display section

Four rows. The first two are constants that describe how the frame reaches the screen; the last two are
read live from the kernel (`src/about/section_render/display.rs:25`):

| Row | Value | Source |
|---|---|---|
| Backend | `compositor + driver.virtio_gpu` | `display.rs:34` |
| Format | `ARGB8888` | `display.rs:35` |
| Width (px) | the primary display width, or `unavailable` | `display.rs:36`, `data/display.rs:19` |
| Height (px) | the primary display height, or `unavailable` | `display.rs:37`, `data/display.rs:19` |

Width and height come from `nonos_display_dimensions(0, ...)`, the syscall that reads the primary
display's size. If the call returns a negative status or a zero dimension, both fields render as
`unavailable` (`src/about/data/display.rs:22`).

### Uptime section

Five rows derived from a single wall-clock read (`src/about/section_render/uptime.rs:25`):

| Row | Value | Source |
|---|---|---|
| Wall ms | the raw milliseconds, or `unavailable` | `uptime.rs:41`, `data/uptime.rs:19` |
| Days | milliseconds split into whole days | `uptime.rs:42`, `data/uptime.rs:27` |
| Hours | remaining whole hours | `uptime.rs:43`, `data/uptime.rs:27` |
| Minutes | remaining whole minutes | `uptime.rs:44`, `data/uptime.rs:27` |
| Seconds | remaining whole seconds | `uptime.rs:45`, `data/uptime.rs:27` |

The raw value is `mk_time_millis()`; a negative return means the clock is unreadable, in which case Wall
ms shows `unavailable` and the split fields show zero (`src/about/data/uptime.rs:19`). Each paint reads the
clock fresh, so the numbers advance as the section is repainted (`src/about/section_render/uptime.rs:31`).

### License section

The full license, with three header rows, a blank line, and the AGPL-3 text line by line
(`src/about/section_render/license.rs:28`):

| Row | Value | Source |
|---|---|---|
| Name | `GNU Affero General Public License` | `license.rs:33`, `data/license.rs:17` |
| Version | `v3 or later (AGPL-3.0)` | `license.rs:33`, `data/license.rs:18` |
| URL | `https://www.gnu.org/licenses/agpl-3.0.html` | `license.rs:33`, `data/license.rs:19` |
| (license body) | every line of the repository `LICENSE` file | `license.rs:46`, `data/license.rs:21` |

The body is the actual `LICENSE` file from the repository root, embedded at compile time with
`include_str!` and split into one row per line, so this is the longest section and the reason the
scrollbar exists (`src/about/data/license.rs:21`, `license.rs:24`).

## Keybindings and interaction

Input arrives as key-down events and pointer button-down events. The manifest subscribes to key-down,
button-down, and absolute-pointer notifications, and the router dispatches a button-down to the tab-strip
handler and a key-down by key code (`src/about/manifest.rs:22`, `src/about/event/router.rs:34`). Anything
that is not one of these is ignored.

| Key | Action | Source |
|---|---|---|
| Tab | select the next section (wraps), reset scroll | `router.rs:44`, `on_tab.rs:21`, `state.rs:36` |
| Shift+Tab | select the previous section (wraps), reset scroll | `router.rs:43`, `on_shift_tab.rs:21`, `state.rs:41` |
| Up | scroll the body up one line | `router.rs:45`, `on_arrow_up.rs:21`, `state.rs:46` |
| Down | scroll the body down one line (clamped to the last page) | `router.rs:46`, `on_arrow_down.rs:22`, `state.rs:49` |
| Page Up | scroll up by a visible page | `router.rs:47`, `on_page_up.rs:21`, `state.rs:55` |
| Page Down | scroll down by a visible page (clamped) | `router.rs:48`, `on_page_down.rs:22`, `state.rs:58` |
| Home | jump to the top of the section | `router.rs:49`, `on_home.rs:21` |
| End | jump to the last page of the section | `router.rs:50`, `on_end.rs:22` |
| Esc | close the window | `router.rs:42`, `on_esc.rs:19` |

A pointer button-down inside the tab strip selects the tab under the cursor. The handler walks the same
tab layout the painter uses (14px padding, an 8px cell per label character) and, if the click lands on a
tab that is not already selected, switches to it and resets the scroll; a click on the current tab or
outside the strip does nothing (`src/about/event/on_pointer_button.rs:27`, `src/about/paint/tabs.rs:27`).

Every handler that changes something returns `Repaint`; the ones that would be no-ops (Home when already
at the top, End when already at the bottom, clicking the current tab) return `Idle` so the runtime does
not repaint needlessly (`src/about/event/on_home.rs:22`, `on_end.rs:25`, `on_pointer_button.rs:44`). The
number of visible body lines is recomputed from the window height on every paint, so scrolling stays
correct if the window is resized (`src/about/state.rs:64`, `src/about/paint/frame.rs:28`).

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` calls the skeleton's `run(About::new)`
(`src/main.rs:27`). The `about` module has one submodule per concern: `app` (the `App` impl), `data` (the
compile-time facts and the two live reads), `event` (the input handlers and router), `format` (a `u64`
to-decimal helper), `manifest` (the window descriptor), `paint` (the renderer), `section` and
`section_render` (the five sections and how each is drawn), `state` (the selection and scroll model), and
`theme` (colors and geometry) (`src/about/mod.rs:17`).

The model is a `State`: the selected section, the scroll offset, a painted flag, and the last computed
count of visible body lines (`src/about/state.rs:22`). It starts on Identity with zero scroll
(`src/about/state.rs:30`).

Lifecycle:

1. The kernel spawns the capsule at boot through the desktop app plan, which runs it second in the app
   fleet after the input proof (`src/userspace/init/spawn_plan/apps.rs:19`,
   `apps.rs:43`). The spawn helper verifies the embedded ELF, id cert, manifest, and attestation trailer
   against the baked trust anchor, registers `app.about` on port 4710 with the reply inbox on 4711, and on
   success logs `[APP-ABOUT] capsule spawned` (`src/userspace/capsule_about/spawn.rs:36`,
   `src/userspace/init/spawn_plan/boot.rs:20`, `src/userspace/init/capsule_boot/run.rs:29`).
2. The skeleton `run` initializes the heap, discovers its peers, then waits for the first delivery and
   builds the `About` app. It creates the window from the manifest (a 560x400 Normal window titled
   `About NONOS`) and drives the event and paint loop (`userland/app_skeleton/src/runner/entry.rs:31`,
   `src/about/manifest.rs:28`, `src/about/theme.rs:27`).
3. Each key-down and tab click flows through `on_event` to the `State`; a handler mutates the selection or
   the scroll and returns `Repaint`, `Idle`, or `Close` (`src/about/event/router.rs:34`).
4. `paint` clears the background, then draws the header, the tab strip, the selected section's body, the
   scrollbar, and the status bar into the shared surface the compositor presents
   (`src/about/paint/frame.rs:27`).

## Protocol and IPC

The about app exposes no application opcodes of its own beyond what the app skeleton registers for it
(the `app.about` service on port 4710 and the reply inbox on 4711,
`src/userspace/capsule_about/spawn.rs:30`). All window registration, paint-buffer allocation, input
delivery, and surface presentation go through the skeleton and the compositor, not through calls the app
makes itself.

Everything the app does on its own is two direct syscalls through `nonos_libc`, both read-only:

```
  N_GFX_DISPLAY_DIMENSIONS   nonos_display_dimensions(0, &w, &h)   the Display section's width/height
  N_MK_TIME_MILLIS           mk_time_millis()                      the Uptime section's clock
```

`nonos_display_dimensions` is the `GraphicsDisplayQuery` syscall; it returns a negative status or zeroed
dimensions when the display is not ready, which the app renders as `unavailable`
(`userland/libc/src/graphics/display_dimensions.rs:20`, `src/about/data/display.rs:22`). `mk_time_millis`
returns the wall-clock milliseconds as an `i64`, negative on failure
(`userland/libc/src/time/wall.rs:19`, `src/about/data/uptime.rs:19`). Neither call carries a request body
or a reply beyond the return register; there is no service handshake and no marshaling.

## Security analysis

The about app is the smallest useful capsule in the tree and a clean example of least privilege. Its mask
`0x1819` grants CoreExec, IPC, Memory, GraphicsDisplayQuery, and GraphicsSurfaceCreate and nothing else
(`Capsule.mk:11`, `src/userspace/capsule_about/spawn.rs:49`). There is no Network bit, no FileSystem bit,
and no Hardware, Driver, Mmio, Irq, Dma, or Pio capability. The app cannot open a socket, read a file,
touch a device register, or map DMA. It reads two scalars from the kernel and paints text.

The two live reads are both queries with no side effects. `GraphicsDisplayQuery` returns the display size;
it does not let the app change the mode, map a framebuffer, or present to scanout. The wall-clock read
returns a number. Neither grants any authority beyond learning a value, and the app holds no capability
that would let it act on those values against any other subsystem.

The app does not even hold `GraphicsSurfaceMap` or `GraphicsPresent`. It holds only
`GraphicsSurfaceCreate`, the right to register its paint surface; the runtime and the compositor own the
mapping and the scanout flush. So a bug in a section renderer or a scroll handler is confined to the app's
own paint buffer and its own `State`. There is no parser for untrusted input, no file path handling, and
no network endpoint. The only external inputs are the key and pointer events the runtime delivers, and the
worst a malformed event can do is get ignored by the router (`src/about/event/router.rs:38`).

The Authority section is worth calling out as a feature rather than a risk: the app deliberately publishes
its own mask and the full denied list so a user can read exactly what the capsule can and cannot reach
(`src/about/section_render/authority.rs:31`). That table is the app's own copy of the capability
descriptors; the authoritative grant is the kernel's spawn request, and the two agree by construction
because both derive from the same five capabilities (`src/about/data/caps.rs:47`, `spawn.rs:49`).

Isolation from other capsules is the kernel's, not the app's: it is a CPL 3 user binary that only speaks
IPC and its own surface, and it is verified and enrolled at spawn like every other capsule
(`src/userspace/capsule_about/spawn.rs:56`).

## How to contribute

The source lives at `userland/capsule_about/`. The compile-time facts are under `src/about/data/`, the
renderers under `src/about/section_render/`, the input handlers under `src/about/event/`, the drawing
under `src/about/paint/`, and the selection and scroll model in `src/about/state.rs`.

To change what a section shows, edit its data module and its renderer together. For example, to change the
tagline, edit `product::TAGLINE` (`src/about/data/product.rs:18`); it is used by both the header and the
Identity section. The Identity, Display, and Uptime sections have a fixed row count declared as
`LINE_COUNT`, so if you add or remove a row you must update that constant to match, or the scrollbar and
the End key will be off (`src/about/section_render/identity.rs:22`, `display.rs:23`, `uptime.rs:23`). The
Authority and License sections compute their line count at runtime, so they need no manual count
(`src/about/section_render/authority.rs:25`, `license.rs:24`).

To add a whole new section:

1. Add a variant to `Section`, extend the `SECTIONS` array, and give it a title and an index
   (`src/about/section.rs:18`).
2. Add a renderer module under `src/about/section_render/` exposing `render(scroll, visible, top, fb)` and
   either a `const LINE_COUNT` or a `fn line_count()`, then wire both into the two match arms in
   `src/about/section_render/mod.rs:28` and `:38`.
3. Draw label/value rows with `row::pair` and full-width rows with `row::single`; they handle the column
   layout and the per-line y offset for you (`src/about/section_render/row.rs:24`).

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk`, included
through `userland/capsule_about/Capsule.mk:14`):

```
  make nonos-mk-about                build the capsule ELF
  make nonos-mk-about-sign           produce the id cert, manifest, and attestation trailer
  make nonos-mk-about-verify         verify the signed artifacts against the trust anchor
  make nonos-mk-check-about-keys     check the per-capsule signing keys exist
```

For a running desktop that includes the about app, `make nonos-mk-about-prod` builds the full desktop GUI
image (`Makefile:1162`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (the two live reads return `Option`/status and fall back to `unavailable` rather
than panicking, and the release profile is `panic = "abort"`, `Cargo.toml:29`); modular files, one unit
per file, with `mod.rs` used only for re-exports (`src/about/mod.rs`, `src/about/data/mod.rs`); and the
AGPL header at the top of every source file, matching the header on every existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints `[APP-ABOUT]
capsule spawned` (tag `APP-ABOUT`, message `capsule spawned`) from the boot log
(`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line means the
capsule never started, usually a signature, manifest, or capability failure; the error path prints an
`[ERROR]` line built from the spawn error instead (`src/userspace/init/capsule_boot/run.rs:32`).

Failure modes and where to look:

- Window opens but the tabs do nothing. The window subscribes to key-down, button-down, and
  absolute-pointer events and ignores everything else (`src/about/manifest.rs:25`,
  `src/about/event/router.rs:38`). If keys and clicks do nothing, the input path into the app (compositor,
  wm, input_router) is the suspect, not the app.
- The Display section reads `unavailable`. `nonos_display_dimensions` returned a negative status or a zero
  dimension, which happens before the display is up or if the query capability is missing
  (`src/about/data/display.rs:22`). Confirm the Authority section lists `GraphicsDisplayQuery` as granted.
- The Uptime section reads `unavailable` or stays at zero. `mk_time_millis` returned negative, so the wall
  clock is unreadable; the split fields then show zero (`src/about/data/uptime.rs:19`).
- Scrolling stops short or overshoots. The visible line count is recomputed from the window height each
  paint and the fixed-count sections carry a hand-maintained `LINE_COUNT`; a mismatch there is the usual
  cause of a scrollbar that does not reach the end (`src/about/state.rs:64`,
  `src/about/section_render/identity.rs:22`).

The app has no serial markers of its own; its only kernel-visible sign of life is the `[APP-ABOUT]
capsule spawned` boot line, after which everything is on-screen.

## Source map

```
  src/main.rs                              _start -> run(About::new)
  src/about/app.rs                         the App impl: manifest, on_event, paint
  src/about/manifest.rs                    the window descriptor (title, size, input mask)
  src/about/state.rs                       State: selected section, scroll, visible-line model
  src/about/section.rs                     the five sections and their order
  src/about/section_render/                the per-section renderers (identity, authority, display, uptime, license)
  src/about/section_render/row.rs          the label/value and single-line row layout
  src/about/data/                          compile-time facts (product, build, abi, caps, trust, license) + the two live reads (display, uptime)
  src/about/event/                         the input router and per-key handlers
  src/about/paint/                         header, tab strip, body, scrollbar, status bar, and the frame compositor
  src/about/format.rs                      u64-to-decimal for the rendered numbers
  src/about/theme.rs                       colors and window geometry
  build.rs                                 stamps the git SHA into ABOUT_GIT_SHA
  Capsule.mk                               slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_about/             the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/apps.rs    the desktop-fleet spawn entry
  userland/libc/src/graphics/display_dimensions.rs   the display-size syscall
  userland/libc/src/time/wall.rs           the wall-clock syscall
  nonos-mk/capsule.mk                      the generated nonos-mk-about[-sign|-verify] targets
```

Every reference above is verified against those trees.

# compositor (full reference)

`compositor` owns the screen. It is the one capsule in the desktop fleet that puts pixels on the display:
every window's surface, the wallpaper, the login prompt, and the boot splash all become visible only
because the compositor composites them into the framebuffer and presents the frame. Client capsules submit
layers (a surface handle and a rectangle on the display) and commit damage; the compositor accumulates the
damaged region, composites the visible layers z-ordered into its backing framebuffer on each vsync, and
presents. It is passive: it receives requests, paints, and calls no capsule except the graphics driver. It
is the capsule that actually presents the [surfaces](../../subsystems/graphics/surfaces.md) every window
draws into. This is the exhaustive reference; the [surfaces](../../subsystems/graphics/surfaces.md) and
[graphics](../../subsystems/graphics/README.md) subsystem pages give the wider picture.

The kernel spawns it under service handle `compositor` on service port 4310 with a reply port on 4311. Its
`Capsule.mk` capability mask is `0x7919`, but the kernel grants it `0x7819` at spawn (the difference is
explained below). The source is `userland/compositor/`.

## Contents

- [Overview and role](#overview-and-role)
- [Identity](#identity)
- [The server loop and frame pacing](#the-server-loop-and-frame-pacing)
- [Operations reference](#operations-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [The scene, damage, and cursor state](#the-scene-damage-and-cursor-state)
- [The compositing pipeline](#the-compositing-pipeline)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## Overview and role

The compositor is the single presenter. Every other desktop capsule that wants to be seen hands it a
surface handle and a rectangle, and the compositor is what turns that into light on the panel. It runs a
damage-driven frame loop: it drains a batch of client requests, and if any damage has accumulated it
composites exactly the dirty region and presents, then blocks until the next vblank
(`userland/compositor/src/server/runner/entry.rs:26`). Nothing is painted when nothing changed, so an idle
desktop costs a vsync wait and no pixels.

It holds two backends. On a machine with a virtio-gpu driver it composites into the driver's primary
surface and presents through the virtio resource ops (transfer, scanout, flush). On a machine without one
(real UEFI hardware, VirtualBox, VMware) it falls back to a GOP framebuffer path: it registers its own
page-aligned backing surface and presents by asking the kernel to blit it to the UEFI linear framebuffer
(`userland/compositor/src/setup/prime_gop.rs:17`). The choice is made once at startup and the loop is
otherwise identical.

## Identity

Everything the kernel and the service registry need to name and reach the compositor comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `compositor` | `Capsule.mk:5` |
| Service handle | `compositor` | `Capsule.mk:6`, `main.rs:33`, `src/userspace/capsule_compositor/spawn.rs:31` |
| Namespace | `systems.nonos.compositor` | `Capsule.mk:11` |
| Service endpoint | `service:4310:compositor` | `Capsule.mk:12`, `spawn.rs:32` |
| Reply endpoint | `reply:4311:endpoint.compositor.reply` | `Capsule.mk:13`, `spawn.rs:33`, `spawn.rs:34` |
| Capsule.mk capability mask | `0x7919` | `Capsule.mk:14` |
| Granted mask at spawn | `0x7819` | `src/userspace/capsule_compositor/spawn.rs:50` |
| Binary name | `compositor` | `Capsule.mk:9` |
| Kernel mirror | `src/userspace/capsule_compositor` | `Capsule.mk:15` |

Two masks are in play and they differ by one bit, so both are stated. The `Capsule.mk` field
`CAPSULE_REQUIRED_CAPS := 0x7919` is what the build stamps into the signed manifest and uses as the
capability ceiling (`nonos-mk/capsule.mk:71`, `:230`). Decomposed against `src/capabilities/types.rs`:

```
  0x0001  CoreExec                bit()  1        types.rs:56
  0x0008  IPC                     bit()  8        types.rs:59
  0x0010  Memory                  bit() 16        types.rs:60
  0x0100  Debug                   bit() 256       types.rs:64
  0x0800  GraphicsDisplayQuery    bit() 2048      types.rs:67
  0x1000  GraphicsSurfaceCreate   bit() 4096      types.rs:68
  0x2000  GraphicsSurfaceMap      bit() 8192      types.rs:69
  0x4000  GraphicsPresent         bit() 16384     types.rs:70
  ------
  0x7919  = 1 + 8 + 16 + 256 + 2048 + 4096 + 8192 + 16384
```

The kernel spawn path, however, requests only seven of those eight, dropping `Debug`
(`src/userspace/capsule_compositor/spawn.rs:50`):

```
  CoreExec | IPC | Memory
  | GraphicsDisplayQuery | GraphicsSurfaceCreate | GraphicsSurfaceMap | GraphicsPresent
  = 1 + 8 + 16 + 2048 + 4096 + 8192 + 16384
  = 0x7819
```

So the effective granted mask is `0x7819`. This is the value the kernel prints in the `[SPAWN]` marker
(the log takes the requested caps bits, `install/install.rs:45`, `:52`) and the value the attestation
inventory records for the compositor (`userland/capsule_attest/src/server/handlers/proof_capsule_list.rs:31`
lists `(b"compositor", 0x7819)`). The `Debug` bit in the manifest ceiling is a headroom bit the running
capsule never asks for. What matters either way is `GraphicsPresent`: the compositor is the only capsule
allowed to put a frame on the display, and `GraphicsSurfaceMap` is what lets it map a client's surface so
it can composite it. There is no `Network`, no `FileSystem`, and none of the driver-broker bits (`Driver`,
`Mmio`, `Irq`, `Dma`, `Pio`).

## The server loop and frame pacing

`_start` initialises the heap, waits for setup, registers the `compositor` service on port 4310, and enters
the loop (`main.rs:37`). Setup (`wait_for_setup::wait_for_setup`, `src/wait_for_setup.rs:25`) tries the
virtio-gpu path first and only falls back to GOP after `VIRTIO_ATTEMPTS_BEFORE_GOP = 6` failed attempts, so
a machine that has virtio-gpu always uses it and only hardware without it takes the GOP route
(`src/wait_for_setup.rs:23`). The loop (`src/server/runner/entry.rs:23`) is:

```
  run(ctx):
      loop:
          drain_ipc(ctx, rx, tx)     // batch up to 16 requests, RECV_NOWAIT
          frame_pacer::tick(ctx)     // if damage is pending, composite and present one frame
          frame_pacer::wait_for_vsync()   // block to the next vblank, or yield on error
```

The loop is damage-driven: it drains a batch of requests, paints only when the accumulator has damage, and
paces to vsync. Draining in batches of up to 16 (`MAX_BATCH`, `src/server/runner/drain.rs:26`) keeps a
burst of scene submissions from starving the frame, and `drain_ipc` uses `RECV_NOWAIT` so an empty inbox
returns immediately (`src/server/runner/drain.rs:25`, `:38`). If `tick` returns an error it is latched once
into `ctx.scanout_error_reported` and the loop keeps running (`src/server/runner/entry.rs:30`); if the
vsync wait fails the loop yields instead of spinning (`entry.rs:35`).

## Operations reference

The compositor exposes eight operations (`src/protocol/ops.rs`), dispatched in
`src/server/runner/dispatch.rs:31`. Each request is the 20-byte `NCMP` header plus a fixed-size payload;
the reply is the header plus a four-byte status, and `DISPLAY_INFO` adds a 16-byte data block. Request
lengths come from `src/protocol/limits.rs`.

| Op | Code | Request payload | Handler | What it does |
|---|---|---|---|---|
| `HEALTHCHECK` | `0x0001` | empty | `handlers/health.rs:20` | reply status 0, liveness probe |
| `SCENE_SUBMIT` | `0x0002` | 32 B | `handlers/scene_submit.rs:21` | register or replace the caller's layer, accumulate its rect as damage |
| `DAMAGE_COMMIT` | `0x0003` | 16 B | `handlers/damage_commit.rs:21` | expand the damage bounding box by a rectangle |
| `FOCUS_SET` | `0x0004` | 8 B | `handlers/focus_set.rs:21` | record which pid holds keyboard focus |
| `INPUT_SUBSCRIBE` | `0x0005` | empty | `handlers/input_subscribe.rs:21` | mark the caller focused (see honest gaps) |
| `CURSOR_UPDATE` | `0x0006` | 16 B | `handlers/cursor_update.rs:22` | move or hide the cursor, damage old and new positions |
| `SCENE_REMOVE` | `0x0007` | 8 B | `handlers/scene_remove.rs:21` | drop all layers owned by the caller and forget their surfaces |
| `DISPLAY_INFO` | `0x0008` | empty | `handlers/display_info.rs:23` | return width, height, stride, pixel format |

`SCENE_SUBMIT` carries `surface_handle (u64), x, y, w, h, z (u32 x5)` (`limits.rs:19` sets 32 bytes). The
handler validates `w > 0`, `h > 0`, and that `x + w` and `y + h` fit inside the display, rejecting with
`E_INVAL` otherwise (`scene_submit.rs:49`). On success it builds a `Layer` owned by the sender pid and
calls `SceneTable::submit`, which replaces the sender's existing layer if it has one or takes a free slot,
then accumulates the layer rectangle as damage (`scene_submit.rs:56`, `:70`). A submit past the 32-slot
table returns `E_INVAL` (`state/scene/submit.rs:28`, surfaced at `scene_submit.rs:67`).

`DAMAGE_COMMIT` carries `x, y, w, h (u32 x4)` and merges the rectangle into the damage box after the same
in-display validation (`damage_commit.rs:43`). `CURSOR_UPDATE` carries `x, y, visible (u32 x3)`; it damages
both the previous cursor cell (if it was visible) and the new one, each a `CURSOR_SIDE = 32` box clipped to
the screen (`cursor_update.rs:20`, `:45`), and it does not send a status reply, it just returns `Ok`
(`cursor_update.rs:30`). `SCENE_REMOVE` collects the caller's surface handles, drops the caller's layers,
accumulates their union rectangle as damage, and releases each surface from the attach cache
(`scene_remove.rs:37`, `:43`, `:46`). `FOCUS_SET` stores a target pid, `INPUT_SUBSCRIBE` marks the sender
itself focused (`focus_set.rs:34`, `input_subscribe.rs:27`), and `DISPLAY_INFO` returns
`width, height, stride, SURFACE_FORMAT_ARGB8888` (`display_info.rs:29`).

An unknown op with an empty body is `E_BAD_OP` and any other malformed request is `E_INVAL`
(`dispatch.rs:44`).

## Architecture and lifecycle

The capsule is `no_std`/`no_main` (`main.rs:17`). Its top-level modules are `frame_pacer` (the compositing
and present pipeline), `gfx_client` (the virtio-gpu IPC client), `protocol` (the wire format), `server`
(the request loop, dispatch, and handlers), `setup` and `wait_for_setup` (display bring-up), `state` (scene,
damage, focus, cursor, attach), and `sw_blitter` (the software blitter) (`main.rs:22`).

Lifecycle:

1. The kernel spawns the compositor as part of the desktop fleet's display core, before the driver and
   network fleets, so the boot splash appears immediately after loader hand-off
   (`src/userspace/init/spawn_plan/desktop_fleet.rs:37`, `:87`). The spawn verifies the embedded ELF, id
   cert, manifest, and attestation, requests the seven caps, registers `compositor` on port 4310, and logs
   `[COMPOSITOR] capsule spawned` (`src/userspace/capsule_compositor/spawn.rs:40`,
   `src/userspace/init/capsule_boot/run.rs:29`).
2. `wait_for_setup` brings up the display and returns a fully populated `Context`. The virtio path looks up
   `driver.virtio_gpu0`, fetches its primary surface, attaches it, validates stride and byte length, and
   marks the whole screen damaged (`src/setup/prime_once.rs:25`, `:52`). The GOP path allocates a
   page-aligned mmap backing, registers it as a surface, self-attaches to get the same VA, and marks full
   damage (`src/setup/prime_gop.rs:36`). Either way the first frame repaints the entire display.
3. `_start` registers the service and enters `server::run`, the loop above (`main.rs:42`).
4. On each iteration the loop drains client requests into scene and damage state, and the frame pacer
   composites and presents any pending damage, paced to vsync.

The `Context` (`src/state/context.rs:19`) is the whole runtime state: the graphics port and resource id,
the display geometry (`width`, `height`, `stride`, `backing_len`, `backing_va`), the `gop_mode` flag, the
present surface handle, the request-id counter, and the five compositing structures (`scene`, `damage`,
`focus`, `cursor`, `attach`).

## The scene, damage, and cursor state

The scene table (`src/state/scene/table.rs:19`) is a fixed array of up to `MAX_LAYERS = 32` layers
(`src/state/scene/layer.rs:17`):

```
  struct Layer { owner_pid, surface_handle, x, y, width, height, z, in_use, miss_count }
```

Layers are stored unsorted. `submit` keys on `owner_pid`, so a second submit from the same pid updates its
one layer rather than adding a new one (`src/state/scene/submit.rs:22`). `z_sorted_snapshot` copies the
in-use layers out and insertion-sorts them by `z`, so a higher-z layer paints on top
(`src/state/scene/snapshot.rs:21`). `drop_by_pid` and `remove_by_pid` clear a pid's layers and return the
union rectangle to damage (`src/state/scene/drop_by_pid.rs:21`, `src/state/scene_remove.rs:23`).

The `DamageAccumulator` (`src/state/damage.rs:30`) is a single bounding rectangle plus a `pending` flag:
`accumulate` merges a rectangle into the box, `drain` returns and clears it, and `mark_full` dirties the
whole display for the first frame (`damage.rs:45`, `:61`, `:40`). The `pending` flag distinguishes "no
damage" from "fully damaged" without a sentinel rect (`damage.rs:17`).

The `CursorTracker` (`src/state/cursor.rs:24`) holds one `CursorState { x, y, visible }` and `update`
returns the previous state so the handler can damage where the cursor was (`cursor.rs:33`). It starts at
the screen centre (`prime_once.rs:70`, `prime_gop.rs:83`). The `FocusTable` (`src/state/focus.rs:22`) just
records a focused pid.

The `AttachCache` (`src/state/attach.rs:31`) is a 32-slot map from a surface handle to a mapped `Surface`.
`get_or_attach` returns a cached mapping or calls `mk_surface_attach` to map the client's surface and caches
the result (`attach.rs:39`); `forget` calls `mk_surface_release` and clears the slot (`attach.rs:63`). This
is where `GraphicsSurfaceMap` is exercised.

## The compositing pipeline

`frame_pacer::tick` (`src/frame_pacer/tick.rs:23`) drains the damage rectangle, and if there is none it
returns without touching the display. If there is, it composites, issues a release fence so the pixel
writes are ordered before the present, and presents:

```
  tick(ctx):
      rect = damage.drain()  or return Ok       tick.rs:24
      composite::paint(ctx, rect)               tick.rs:27
      fence(Release)                            tick.rs:28
      if gop_mode: mk_surface_present(handle)   tick.rs:29
      else: transfer_to_host; (first frame) set_scanout; resource_flush   tick.rs:37..72
```

`composite::paint` (`src/frame_pacer/composite.rs:26`):

```
  paint(ctx, rect):
      fill rect with BACKGROUND_ARGB (a dark blue-grey 0xFF101620)   composite.rs:34
      for layer in scene.z_sorted_snapshot():
          src = attach.get_or_attach(layer.surface_handle)           composite.rs:39
          sw_blitter::composite_layer(dst, src, layer rect, clip=rect)  composite.rs:40
      reap layers missing REAP_THRESHOLD = 60 consecutive frames      composite.rs:57
      if cursor.visible: draw the cursor sprite clipped to rect       composite.rs:62
```

The blitter is pure software (`src/sw_blitter/`). `fill_rect` clips the rectangle to the surface and fills
whole rows (`src/sw_blitter/fill.rs:20`). `composite_layer` clips the layer rectangle to both the
destination and the damage rectangle, then copies pixels column by column, writing only where the source
alpha is non-zero, so transparent client pixels do not overwrite what is under them
(`src/sw_blitter/copy_rect.rs:20`, `:59`). Every read and write goes through `Surface::row_start`, which
bounds-checks the row against the surface's declared `byte_len` before yielding a pointer
(`src/sw_blitter/mod.rs:33`). The cursor is a 14-pixel white arrow with a one-pixel shadow, drawn with
volatile writes and clipped to the damage rect (`src/frame_pacer/cursor.rs:24`).

Presentation is one of two backends. In GOP mode the composed pixels already live in the registered
surface, so the compositor calls `mk_surface_present(surface_handle)` and the kernel blits that surface to
the UEFI framebuffer (`tick.rs:32`). In virtio-gpu mode the compositor computes the pixel offset of the
dirty rect, transfers just that region to the device (`transfer_to_host`), sets the scanout on the very
first frame only (`first_scanout_done`), and flushes the resource (`resource_flush`) (`tick.rs:37`, `:49`,
`:63`).

## Protocol and IPC

The compositor speaks two wire protocols: its own inbound `NCMP` protocol that clients call, and an
outbound `NVGP` protocol it uses to drive the virtio-gpu driver. It calls no other capsule.

The inbound frame is `NCMP` (`src/protocol/header.rs:20`): magic `0x4E43_4D50`, version 1, a 20-byte header
(`magic, version, op, flags, reserved, request_id, payload_len`) and a payload capped at
`IPC_PAYLOAD_MAX = 256` bytes. `parse` rejects a short buffer with `E_BAD_LEN`, a wrong magic with
`E_BAD_MAGIC`, a wrong version with `E_BAD_VERSION`, and any header whose `payload_len` does not exactly
match the received length with `E_BAD_LEN` (`src/protocol/decode.rs:19`). The reply reuses the request's op,
flags, and request_id, and carries a signed little-endian status word; `DISPLAY_INFO` follows the status
with its 16-byte data block (`src/protocol/encode.rs:19`, `src/server/respond.rs:32`). Error codes are
`E_INVAL -22`, `E_BAD_OP -38`, `E_BAD_MAGIC -71`, `E_BAD_LEN -90`, `E_BAD_VERSION -93`
(`src/protocol/errno.rs`).

Clients call the compositor through the app skeleton and the desktop SDK, resolving it by
`mk_service_lookup("compositor")`:

- Every GUI app, through the app skeleton, calls `SCENE_SUBMIT`, `DAMAGE_COMMIT`, `SCENE_REMOVE`,
  `DISPLAY_INFO`, and `HEALTHCHECK` (`userland/app_skeleton/src/clients/compositor/mod.rs:23`).
- The input router sends `CURSOR_UPDATE` and queries `DISPLAY_INFO`
  (`userland/capsule_input_router/src/clients/compositor/constants.rs:18`,
  `.../cursor_update.rs:39`, `.../display_size.rs:25`).
- The wm, wallpaper, login, desktop shell, boot splash, and setup wizard all resolve `compositor` at
  startup and submit their own layers (`userland/capsule_wm/src/setup/discover.rs:19`,
  `userland/capsule_boot_splash/src/main.rs:78`, and the sibling capsules).

The outbound `NVGP` protocol drives `driver.virtio_gpu0`, which the compositor resolves once at setup
(`src/setup/discover.rs:19`, `:28`). Its wire is a separate 20-byte header, magic `0x4E56_4750`, version 1
(`src/gfx_client/wire.rs:24`). The ops the compositor issues:

```
  0x000C  GET_PRIMARY_SURFACE   fetch handle, resource_id, geometry, format   gfx_client/get_primary.rs:22
  0x0008  TRANSFER_TO_HOST      copy the dirty rect to the device             gfx_client/transfer.rs:21
  0x0009  SET_SCANOUT           bind the resource to scanout 0 (first frame)  gfx_client/set_scanout.rs:21
  0x000A  RESOURCE_FLUSH        flush the dirty rect to the panel             gfx_client/flush.rs:21
```

Each virtio call checks the reply status and returns a specific `&'static str` on a non-zero driver status
(for example `"gfx transfer: driver rejected"`, `transfer.rs:44`). The `GET_PRIMARY_SURFACE` call at setup
uses a shorter boot timeout; the per-frame calls use a 1000 ms call timeout
(`src/gfx_client/wire.rs:27`, `:28`).

The compositor also uses kernel surface primitives directly: `mk_surface_attach` and `mk_surface_release`
for the attach cache, `mk_surface_register` and `mk_mmap` for the GOP backing, `mk_surface_present` for the
GOP present, and `mk_display_vsync_wait` for pacing (`src/state/attach.rs:17`, `src/setup/prime_gop.rs:24`,
`src/frame_pacer/vsync.rs:17`).

## Security analysis

The compositor looks powerful because it is the one capsule that sees every window's pixels, but its
authority is narrow and framebuffer-bounded. The granted mask is `0x7819`
(`src/userspace/capsule_compositor/spawn.rs:50`): CoreExec, IPC, Memory, and the four graphics caps. The
one that matters is `GraphicsPresent`, the sole right to present a frame, and `GraphicsSurfaceMap`, the
right to map a client surface so it can be composited. It holds none of the driver-broker capabilities: no
`Driver`, `Mmio`, `Irq`, `Dma`, or `Pio`. It never touches a device register or programs DMA. It presents
through the kernel (GOP mode) or through the virtio-gpu driver over IPC (virtio mode), so a compromise of
the compositor is bounded to what lands in the framebuffer, not the GPU's registers or memory.

- **Layers are owner-scoped.** A layer is tagged with the submitting pid, `submit` keys on that pid, and
  `SCENE_REMOVE` only touches the caller's own layers (`src/state/scene/submit.rs:22`,
  `src/server/handlers/scene_remove.rs:34`). One capsule cannot move, replace, or delete another's window.
- **Geometry is validated.** A scene or damage rectangle must fit inside the display or it is rejected with
  `E_INVAL` before any state changes (`scene_submit.rs:49`, `damage_commit.rs:43`), and the cursor position
  is bounds-checked (`cursor_update.rs:42`). A client cannot address pixels outside the screen.
- **The wire is strictly parsed.** Magic, version, and an exact-length payload are all checked before
  dispatch, and every handler re-checks its own fixed request length (`src/protocol/decode.rs:19`, and each
  handler's leading length guard). A short or malformed frame is a status error, never a read past the
  buffer.
- **Blits are bounds-checked and alpha-gated.** Every source and destination access goes through
  `Surface::row_start`, which validates the span against the surface `byte_len`
  (`src/sw_blitter/mod.rs:33`), and the compositor blits the client's pixels without interpreting them, so
  untrusted surface content cannot drive the compositor beyond copying it.
- **Passive.** The compositor initiates no call to any capsule other than the graphics driver it presents
  through. It is a target, not a client; its inbound attack surface is the eight fixed-size ops.

Isolation between clients is the kernel's, enforced through per-pid surface handles and the owner-scoped
scene table; the compositor never holds more authority than the right to map a surface it was handed and to
present the result.

## How to contribute

The source lives at `userland/compositor/`. The request loop and dispatch are under `src/server/`, the
per-op handlers under `src/server/handlers/`, the compositing and present pipeline under `src/frame_pacer/`,
the scene and damage state under `src/state/`, the software blitter under `src/sw_blitter/`, the virtio-gpu
client under `src/gfx_client/`, and the display bring-up under `src/setup/` and `src/wait_for_setup.rs`.

To change what clients can request, edit the protocol and a handler:

1. Add the opcode to `src/protocol/ops.rs` and, if it has a fixed request length, a constant to
   `src/protocol/limits.rs`, re-exporting both from `src/protocol/mod.rs`.
2. Write the handler as one file under `src/server/handlers/`, taking `(ctx, sender_pid, req, body, tx)`,
   guarding its request length first and replying through `respond::status` or `respond::status_payload`
   (mirror `handlers/damage_commit.rs`). Add its `pub mod` line to `src/server/handlers/mod.rs`.
3. Wire the op into the match in `src/server/runner/dispatch.rs:31`.

To change compositing or pacing, edit `src/frame_pacer/`: `composite.rs` for how layers are drawn (the fill
colour, z-order, alpha rule, reap threshold), `tick.rs` for the present sequence and the two backends, and
`vsync.rs` for how the loop paces. The blitter itself is `src/sw_blitter/`.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`, included
through `userland/compositor/Capsule.mk:17`):

```
  make nonos-mk-compositor              build the capsule ELF
  make nonos-mk-compositor-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-compositor-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-compositor-keys   check the per-capsule signing keys exist
```

There is no `nonos-mk-compositor-prod` target; the compositor ships inside the full desktop image built by
`make nonos-mk-desktop-gui-prod` and `make nonos-mk-full-gui-prod` (`Makefile:1067`, `Makefile:1093`), and
its artifacts are pulled into those builds through `$(compositor_ARTIFACTS)` (`Makefile:880`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns a status or a `Result<(), &'static str>`, never a panic);
modular files, one unit per file, with `mod.rs` used only for re-exports; and the AGPL header at the top of
every source file, matching the header on every existing module.

## Debugging

The first thing to confirm is that the capsule spawned. On a successful boot the kernel prints
`[COMPOSITOR] capsule spawned` (tag `COMPOSITOR`, message `capsule spawned`, `boot.rs:93`,
`src/userspace/init/capsule_boot/run.rs:29`) and, from the install path, a
`[SPAWN] name=compositor pid=0x... caps=0x7819 entry=0x...` line
(`src/kernel_core/process_spawn/capsule_spawn/runner/install/spawn_log.rs:17`). Note `caps=0x7819`, the
granted mask, not the `0x7919` manifest ceiling. An absent `[COMPOSITOR]` line means the capsule never
started, usually a signature, manifest, or capability failure, and the boot log prints an `[ERROR]` line
instead (`capsule_boot/run.rs:32`).

The compositor is the first desktop service the fleet waits on: the boot splash and wallpaper both poll
`mk_service_lookup("compositor")` and retry until it resolves (`capsule_boot_splash/src/main.rs:78`), so a
desktop that never paints usually means the compositor never registered. Because setup acquires the graphics
port, the backing framebuffer, its geometry, and the GOP-versus-virtio mode before the loop runs, a
compositor that spawns but never presents is stuck in that acquisition (`src/wait_for_setup.rs:25`), not the
loop.

Failure modes and where to look:

- **Black screen, nothing paints.** Either setup never completed (no framebuffer acquired) or nothing has
  ever been damaged. The first real frame comes from `mark_full` at setup (`prime_once.rs:53`,
  `prime_gop.rs:65`), so a black screen after a clean spawn points at a client never submitting a layer,
  not at the compositor.
- **A window never appears.** A `SCENE_SUBMIT` with a zero-area rectangle or one that does not fit the
  display is rejected with `E_INVAL` before a layer is created (`scene_submit.rs:49`), so a missing window
  is usually a geometry rejection, not a lost frame. A submit past 32 live layers also returns `E_INVAL`
  (`submit.rs:28`).
- **A window lingers after its capsule dies, then vanishes.** A layer whose surface can no longer be
  attached is not painted and its `miss_count` climbs; after `REAP_THRESHOLD = 60` consecutive misses the
  layer is reaped and its surface forgotten (`composite.rs:24`, `:57`, `src/state/scene/reap_unattachable.rs:37`).
  A window that hangs for about a second after its owner exits and then disappears is the reaper working.
- **No present in virtio mode.** Each virtio op returns a specific error on a non-zero driver status
  (`gfx transfer/scanout/flush: driver rejected`, `transfer.rs:44`, `set_scanout.rs:44`, `flush.rs:44`). A
  compositing loop that runs but shows nothing on a virtio machine is a driver rejection on transfer,
  scanout, or flush; the first-frame `set_scanout` in particular must succeed or the resource is never
  bound (`tick.rs:49`).
- **No present in GOP mode.** `mk_surface_present` returning negative yields `"gop present rejected"`
  (`tick.rs:32`); this usually means the registered backing surface's VA did not resolve to a real VMA,
  which is exactly why the GOP path backs the surface with a dedicated mmap rather than the heap
  (`prime_gop.rs:98`).
- **A visible smear between two small changes.** Damage is a single bounding box, so a change in two
  opposite corners dirties everything between them and repaints it (`src/state/damage.rs:17`). That is the
  accumulator merging rectangles, expected in v1.

## Honest gaps

Stated from the code:

- `INPUT_SUBSCRIBE` records the caller as focused but v1 does not fan input out through the compositor; the
  [input router](../input-router/README.md) does the fan-out (`src/server/handlers/input_subscribe.rs:27`).
- The pid set by `FOCUS_SET` is recorded but not yet used to highlight a focused window
  (`src/state/focus.rs:31`).
- Damage is a single bounding box rather than a per-tile queue, so a small change in two corners dirties the
  box between them (`src/state/damage.rs:17`).
- The cursor is a software blit, not a hardware sprite (`src/frame_pacer/cursor.rs`).
- The `Debug` bit sits in the `Capsule.mk` manifest ceiling (`0x7919`) but the running capsule is granted
  and attested at `0x7819` (`Capsule.mk:14`, `spawn.rs:50`,
  `userland/capsule_attest/src/server/handlers/proof_capsule_list.rs:31`).

Client surface pixels are untrusted: the compositor blits them but does not interpret them.

## Source map

```
  userland/compositor/src/main.rs                      _start: heap, setup, register, run
  userland/compositor/src/wait_for_setup.rs            virtio-first, GOP-fallback bring-up
  userland/compositor/src/setup/{prime_once,prime_gop,discover}.rs   the two display backends
  userland/compositor/src/server/runner/{entry,drain,dispatch}.rs    the loop, batch drain, op dispatch
  userland/compositor/src/server/handlers/             the eight op handlers
  userland/compositor/src/server/respond.rs            status and status+payload replies
  userland/compositor/src/protocol/{ops,header,limits,decode,encode,errno}.rs   the NCMP wire
  userland/compositor/src/frame_pacer/{tick,composite,cursor,vsync}.rs   compositing and present
  userland/compositor/src/sw_blitter/{mod,fill,copy_rect}.rs   the bounds-checked software blitter
  userland/compositor/src/state/{context,damage,cursor,focus,attach}.rs   the runtime state
  userland/compositor/src/state/scene/                 the 32-slot layer table, submit, snapshot, reap
  userland/compositor/src/gfx_client/                  the virtio-gpu NVGP client
  userland/compositor/Capsule.mk                       slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_compositor/                    the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/desktop_fleet.rs       the desktop-fleet spawn entry
  src/capabilities/types.rs                            the capability bit definitions
  nonos-mk/capsule.mk                                  the generated nonos-mk-compositor[-sign|-verify] targets
```

Every reference above is verified against those trees.
</content>
</invoke>

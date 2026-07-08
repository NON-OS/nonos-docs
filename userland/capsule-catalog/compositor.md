# compositor

`compositor` owns the screen. Client capsules submit layers (a surface handle and a rectangle on the
display) and commit damage; the compositor accumulates the damaged region, composites the visible layers
z-ordered into the framebuffer on each vsync, and presents the frame. It is passive, it receives requests
and paints, and calls no other service except the graphics driver, and it is the capsule that actually
presents the [surfaces](../../subsystems/graphics/surfaces.md) every window draws into. Service
`compositor` on port 4310, capability mask `0x7919`. The source is `userland/compositor/`.

## Contents

- [The server loop and frame pacing](#the-server-loop-and-frame-pacing)
- [The wire protocol](#the-wire-protocol)
- [The operations](#the-operations)
- [The scene, damage, and cursor state](#the-scene-damage-and-cursor-state)
- [The compositing pipeline](#the-compositing-pipeline)
- [Presentation: GOP and virtio-gpu](#presentation-gop-and-virtio-gpu)
- [Layer reaping](#layer-reaping)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop and frame pacing

`main.rs:37` initializes the heap, waits for setup (acquiring the graphics port, the backing framebuffer,
its dimensions and stride, and whether it is in GOP mode or virtio-gpu mode), registers the service, and
runs the loop (`src/server/runner/entry.rs:23`):

```
  run(ctx):
      loop:
          drain_ipc(ctx, rx, tx)           // batch up to 16 requests, RECV_NOWAIT
          frame_pacer::tick(ctx)           // if damage is pending, composite one frame
          frame_pacer::wait_for_vsync()    // block to the next vblank, or yield on error
```

The loop is damage-driven: it drains a batch of requests, paints only when there is accumulated damage,
and paces to vsync. Draining in batches of up to 16 (`src/server/runner/drain.rs:28`) keeps a burst of
scene submissions from starving the frame.

## The wire protocol

The frame is `NCMP` (magic `0x4E43_4D50`), version 1, a 20-byte header (`magic, version, op, flags,
reserved, request_id, payload_len`) and a payload capped at `IPC_PAYLOAD_MAX = 256` bytes; the response
is the header plus a four-byte status. The small payload cap fits the fixed-size scene and damage
requests.

## The operations

Eight operations (`src/protocol/ops.rs`), with their fixed request lengths:

```
  0x1 HEALTHCHECK    -
  0x2 SCENE_SUBMIT   32 B: surface_handle(u64), x, y, w, h, z (u32 x5)
  0x3 DAMAGE_COMMIT  16 B: x, y, w, h (u32 x4)
  0x4 FOCUS_SET      which pid holds keyboard focus
  0x5 INPUT_SUBSCRIBE (empty)
  0x6 CURSOR_UPDATE  16 B: x, y, visible
  0x7 SCENE_REMOVE   remove the caller's layer
  0x8 DISPLAY_INFO   (empty) -> dimensions and mode
```

`SCENE_SUBMIT` (`src/server/handlers/scene_submit.rs:21`) validates that `w, h > 0` and the rectangle
lies inside the display, creates or updates a `Layer` owned by the sender pid, and accumulates its
rectangle as damage. `DAMAGE_COMMIT` expands the damage bounding box, again validated against the display.
An unknown op or a wrong-size payload is `E_BAD_OP` or `E_INVAL`.

## The scene, damage, and cursor state

The `Context` (`src/state/context.rs:19`) holds the graphics resource handles, the display geometry, and
the compositing state: a `SceneTable`, a `DamageAccumulator`, a `FocusTable`, a `CursorTracker`, and an
`AttachCache`. The scene table (`src/state/scene/table.rs`) is a fixed array of up to `MAX_LAYERS = 32`
`Layer`s:

```
  struct Layer { owner_pid, surface_handle, x, y, width, height, z, in_use, miss_count }
```

Layers are stored unsorted and `z_sorted_snapshot` returns them ordered by `z`, so a higher-z layer paints
on top. The `DamageAccumulator` (`src/state/damage.rs:22`) is a single bounding rectangle plus a pending
flag: `accumulate` merges a rectangle into the box, `drain` returns and clears it, and `mark_full` dirties
the whole display. The `CursorTracker` (`src/state/cursor.rs:24`) holds the cursor position and
visibility.

## The compositing pipeline

`frame_pacer::tick` (`src/frame_pacer/tick.rs:23`) drains the damage rectangle and, if any, paints it
(`src/frame_pacer/composite.rs:26`):

```
  composite(ctx, rect):
      fill rect with the background color (a dark teal)
      for layer in scene.z_sorted_snapshot():
          surface = attach.get_or_attach(layer.surface_handle)   // map the client surface
          sw_blitter::composite_layer(surface, into framebuffer at layer rect)
      reap layers whose attach has missed 60 consecutive frames
      if cursor.visible:  blit the cursor sprite
      memory fence (Release)
```

The composite fills the damaged rectangle with the background, blits each visible layer's surface into the
framebuffer in z-order through a software blitter, reaps unattachable layers, and blits the cursor. The
release fence orders the pixel writes before the present.

## Presentation: GOP and virtio-gpu

After compositing, the frame is presented one of two ways depending on how the display was brought up. In
GOP mode (a firmware framebuffer), `mk_surface_present(surface_handle)` hands the frame to the kernel. In
virtio-gpu mode, the graphics client transfers the pixels to the device (`transfer_to_host`), sets the
scanout on the first frame, and flushes the resource (`resource_flush`). Either way the compositor writes
to its backing framebuffer and then triggers the present appropriate to the backend.

## Layer reaping

A layer whose client surface cannot be attached, because the client exited or released its surface, is
not painted, and a `miss_count` is incremented; after 60 consecutive misses the layer is reaped from the
scene table (`REAP_THRESHOLD`). So a crashed client's window disappears on its own rather than lingering
as a dead layer, and the fixed 32-slot table does not fill with stale entries.

## Security analysis

- **Layers are owner-scoped**: a layer is tagged with the submitting pid, and only the owner can update or
  remove it, so one capsule cannot move or delete another's window.
- **Geometry is validated**: a scene or damage rectangle must fit inside the display, so a client cannot
  address pixels outside the screen.
- **Passive**: the compositor initiates no calls to other capsules, so it is a target, not a client, and
  its attack surface is the eight fixed-size ops.

## Honest gaps

Stated from the code: `INPUT_SUBSCRIBE` records a subscriber but v1 does not yet fan out input through the
compositor (the [input router](input-router.md) does the fan-out); the focus set by `FOCUS_SET` is not yet
used to highlight a window; damage is a single bounding box rather than a per-tile queue, so a small change
in two corners dirties the box between them; and the cursor is a software blit, not a hardware sprite. The
client surface pixels are untrusted, the compositor blits them but does not interpret them.

## Source map

```
  userland/compositor/src/server/runner/{entry.rs, drain.rs, dispatch.rs}   the loop and dispatch
  userland/compositor/src/server/handlers/{scene_submit, damage_commit, cursor_update, display_info}.rs
  userland/compositor/src/frame_pacer/{tick.rs, composite.rs}               the compositing pipeline
  userland/compositor/src/state/{context.rs, scene/table.rs, damage.rs, cursor.rs}   the scene state
  userland/compositor/src/gfx_client/                                        the virtio-gpu client
```

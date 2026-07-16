# capsule_wallpaper

The wallpaper capsule renders the desktop background: a color or a catalog image, with a fade animation
on change, painted into a backing surface the compositor presents. Service `wallpaper` on port 4340,
capability mask `0x1819`. The source is `userland/capsule_wallpaper/`.

## The server loop

`main.rs:34` waits for setup (discovering the compositor, catalog, image codec, and policy) and runs the
loop (`src/server/runner/entry.rs:27`): drain IPC, tick the subscriber poll and the fade, and wait for
vsync when idle. The frame is `NWLP` (magic `0x4E574C50`), version 1.

## The operations

Five operations (`src/protocol/ops.rs`):

```
  HEALTHCHECK=1  SET_WALLPAPER=2  GET_WALLPAPER=3  SET_POLICY=4  FADE=5
```

`SET_WALLPAPER` sets the background color and repaints; `SET_POLICY` selects the fit style (stretch,
fit, tile); `FADE` (`src/server/handlers/fade.rs`) starts a `FadeTimeline` that linearly interpolates the
alpha from the current value to a target over a duration, so a wallpaper change dissolves rather than
snapping; `GET_WALLPAPER` returns the current color, policy, dimensions, and alpha.

## State and honesty

The `Context` (`src/state/context.rs:19`) holds the compositor port, the backing surface, the current
color and alpha, the fit policy, the fade timeline, and the catalog port. The fade
(`src/state/fade.rs:22`) samples an eased alpha by elapsed time. It calls the compositor to commit
damage, the catalog to fetch image chunks, the image codec to decode, and policy to read the configured
style. Honest gaps: the catalog and image-codec integration is not fully wired, so in practice the
wallpaper is a solid color or a catalog image, and there is a duplicated JPEG decoder in the wallpaper's
own paint path rather than always going through the [image codec](image-codec.md) service.

## Security analysis

The mask is `0x1819` (`CAPSULE_REQUIRED_CAPS` in `userland/capsule_wallpaper/Capsule.mk`), decoding to
`CoreExec | IPC | Memory | GraphicsDisplayQuery | GraphicsSurfaceCreate` against `src/capabilities/types.rs`.
It has `GraphicsSurfaceCreate` to register its backing surface (`src/setup/prime/register.rs:41`) and
`GraphicsDisplayQuery` to size it. Like the other clients it does not hold `GraphicsPresent`: it paints the
background into its surface and commits damage, and the [compositor](compositor.md) presents. It has no
hardware, filesystem, network, or crypto reach.

- **A background painter, nothing more.** The mask lets the wallpaper draw into one surface it owns and ask
  the compositor to show it. It cannot present directly, cannot map another capsule's surface, and cannot
  reach a device, so a compromise of the wallpaper is bounded to painting garbage into the background layer,
  which the compositor still composites under everything else.
- **Image bytes should be someone else's problem.** The design intent is that untrusted image bytes are
  decoded in the isolated [image codec](image-codec.md) capsule, and the wallpaper only receives a decoded
  surface. That keeps parser-bug exposure out of the wallpaper.
- **Honest boundary: the duplicated JPEG path.** As the state note says, there is a JPEG decoder in the
  wallpaper's own paint path (`src/paint/decode_jpeg.rs`) as well as a `decode_client`
  (`src/decode_client/`) that calls the image codec. When the wallpaper decodes a catalog JPEG in-process it
  runs a real parser over image bytes inside a capsule that also owns the background surface, which is
  exactly the isolation the separate codec exists to provide. The honest posture is that this in-process
  decoder is a duplicate of truth and the path that should win is the codec service.

## Debugging

The wallpaper registers as `wallpaper` on port 4340 and is reached by `mk_service_lookup("wallpaper")`; the
[desktop shell](desktop-shell.md) treats that lookup as required, so the wallpaper must register before the
shell finishes setup. During its own setup it waits for the compositor by name, retrying until the lookup
resolves, and gives up with `"compositor service not announced"` if it never does
(`src/setup/discover.rs:39`). The catalog and policy peers are looked up too but the integration through
them is not fully wired (see the honesty note), so in practice a missing catalog degrades to a solid color
rather than failing setup. The kernel spawn marker is:

```
  [SPAWN] name=wallpaper pid=0x... caps=0x1819 entry=0x...
```

`caps=0x1819` confirms the admitted mask. The runtime failure signatures are on the wire: a `SET_POLICY`
with an unknown fit style or a malformed request is rejected by the handler, and a `FADE` runs its timeline
to completion regardless, so a wallpaper that snaps instead of dissolving is a fade that was started with a
zero duration, not a fault.

## Source map

```
  userland/capsule_wallpaper/src/server/runner/         the loop, drain, vsync wait
  userland/capsule_wallpaper/src/server/handlers/        set_wallpaper, fade, set_policy, get_wallpaper
  userland/capsule_wallpaper/src/setup/discover.rs       compositor wait, "compositor service not announced"
  userland/capsule_wallpaper/src/state/context.rs        color, policy, fade, ports
  userland/capsule_wallpaper/src/state/fade.rs           the fade timeline
  userland/capsule_wallpaper/src/paint/decode_jpeg.rs    the in-process JPEG decoder (duplicate of truth)
  userland/capsule_wallpaper/src/decode_client/          the image-codec client (the path that should win)
  userland/capsule_wallpaper/Capsule.mk                  CAPSULE_REQUIRED_CAPS = 0x1819, endpoint 4340
```

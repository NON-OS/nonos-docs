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

## Source

```
  userland/capsule_wallpaper/src/server/runner/       the loop
  userland/capsule_wallpaper/src/server/handlers/      set_wallpaper, fade, set_policy
  userland/capsule_wallpaper/src/state/context.rs      color, policy, fade, ports
  userland/capsule_wallpaper/src/state/fade.rs         the fade timeline
```

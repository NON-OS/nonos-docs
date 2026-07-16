# Reading and Rendering the Input Diagnostic

This page is the heart of the probe: how a routed input event arrives, how it is decoded and filtered,
and how a printable key becomes pixels. It mirrors `src/server/`, `src/protocol.rs`, and `src/render/`.
For what the probe is and its identity, read the [overview](README.md); for how an event reaches the
probe in the first place, read [the input path](../../subsystems/input/path.md).

## The receive loop

`server::run` (`src/server/runner.rs:12`) is the whole steady state. Before the loop it does two IPC
calls to the router: `subscribe` and then `grab_keyboard` (`src/server/runner.rs:13`,
`src/server/runner.rs:14`). Both are best-effort; their results are discarded, so the probe keeps running
even if the router refuses, and the loop below simply never sees an event. It then allocates a receive
buffer sized `DELIVERY_LEN.max(64)` and blocks (`src/server/runner.rs:15`).

Each iteration calls `mk_ipc_recv_from` on the service inbox with an infinite block
(`src/server/runner.rs:18`). A non-positive return means no message, and the loop continues
(`src/server/runner.rs:25`). Otherwise the received bytes are handed to `parse_delivery`, and a decode
failure is skipped silently (`src/server/runner.rs:28`). Only a `KEY_DOWN` event is acted on; every other
kind is dropped, which is why the probe subscribes to keys only (`src/server/runner.rs:31`).

## Subscribe and grab

Both router calls go through the same helper. `keys_request` builds an 8-byte body whose first four bytes
are the keyboard kind mask `0b11` (`src/clients/input_router.rs:7`, `src/clients/input_router.rs:26`),
wraps it in the `NIRS` request header (`src/protocol.rs:3`, `src/protocol.rs:15`), and sends it with
`mk_ipc_call`. `subscribe` sends it under `OP_SUBSCRIBE` (`0x0002`) and `grab_keyboard` under
`OP_GRAB_REQUEST` (`0x0003`) (`src/protocol.rs:7`, `src/protocol.rs:8`, `src/clients/input_router.rs:32`,
`src/clients/input_router.rs:38`). `call_status` treats a reply shorter than header-plus-status as an
error and reads the `i32` status word after the 20-byte header, returning it verbatim when non-zero
(`src/clients/input_router.rs:11`). The `0b11` mask matches the keyboard bits the router uses to split a
grab from a pointer grab, so the probe claims the keyboard class and nothing else.

## Decoding a delivery

`parse_delivery` (`src/protocol.rs:27`) is the inverse of the router's delivery encoder. It rejects a
buffer shorter than `DELIVERY_LEN`, which is the 8-byte header plus a 32-byte `InputEvent`
(`src/protocol.rs:13`). It checks the magic is `NINP` (`0x4E49_4E50`) and the version is 1, returning
`None` on either mismatch (`src/protocol.rs:10`, `src/protocol.rs:31`). It then reads the eight
`InputEvent` fields little-endian at their fixed offsets: `kind` at 8, `flags` at 10, `code` at 12, `x`
and `y` at 16 and 20, `delta_x` and `delta_y` at 24 and 28, and `timestamp_ns` at 32
(`src/protocol.rs:39`). This is the same 40-byte NINP frame the router emits, so the probe reads exactly
what the router wrote, field for field.

## The printable filter

`on_key` (`src/server/runner.rs:37`) is where a keystroke becomes screen state. It accepts only a `code`
in the printable ASCII range `0x20..=0x7E`; a control key, arrow, or function key is ignored
(`src/server/runner.rs:38`). An accepted code is truncated to a byte, pushed into the history, drawn, and
followed by a compositor damage commit so the frame reaches the display (`src/server/runner.rs:39`,
`src/server/runner.rs:40`). Nothing here interprets `flags`, so a shifted key is rendered as whatever
code the driver and router already resolved.

## The history ring and glyph draw

`push_and_draw` (`src/render/mod.rs:10`) keeps a 64-byte history in the `Context` buffer
(`src/state.rs:9`). When the cursor reaches the buffer length it wraps to zero, so the display holds the
most recent run of keys rather than growing without bound (`src/render/mod.rs:11`). The byte is stored,
the cursor advances, and `redraw` repaints the whole surface (`src/render/mod.rs:19`): it first fills the
surface with the background colour, then walks the history left to right, drawing one glyph per stored
byte with a fixed horizontal advance of `GLYPH_W * SCALE + SCALE` pixels
(`src/render/mod.rs:6`, `src/render/mod.rs:24`).

`draw_glyph` (`src/render/mod.rs:28`) fetches the eight-row bitmap for the character and, for each set
bit in a row, paints a `SCALE`-by-`SCALE` block in the foreground colour (`SCALE` is 4)
(`src/render/mod.rs:6`, `src/render/mod.rs:33`). `font::rows` (`src/render/font.rs:5`) upper-cases the
character, returns the bitmap for `A`-`Z` and `0`-`9` from the tables, a blank for space, and a boxed
fallback glyph for anything else (`src/render/font.rs:7`, `src/render/font_table.rs:1`). The font is a
built-in 8-row bitmap table (`src/render/font_table.rs:11`, `src/render/font_table.rs:40`), so the probe
carries no font asset and cannot render lower-case, punctuation, or non-Latin glyphs; those show as the
fallback box.

## Writing to the surface

`fill_rect` (`src/render/mod.rs:43`) is the only pixel writer. It clamps `xx` below `width` and `yy`
below `height`, computes the byte offset as `yy * stride + xx * 4`, and does a `write_volatile` of the
ARGB word (`src/render/mod.rs:44`). The safety comment records the invariant: `ctx.base` is a mapped
ARGB8888 surface of `stride * height` bytes owned exclusively by the probe, and the clamps keep every
write inside it (`src/render/mod.rs:47`). The initial background fill during setup uses the same layout
through a standalone `fill` (`src/setup/fill.rs:1`). The probe writes straight into its mapped surface and
tells the compositor to present it; it never holds a `Present` or `SurfaceMap` capability, which is why
those are the compositor's and driver's job, not the probe's.

## Source map

```
  userland/capsule_input_probe/src/server/runner.rs   recv loop, subscribe, grab, printable filter
  userland/capsule_input_probe/src/protocol.rs        NIRS request encode, NINP delivery decode
  userland/capsule_input_probe/src/clients/input_router.rs  subscribe and grab IPC clients
  userland/capsule_input_probe/src/render/mod.rs      history ring, redraw, glyph and rect draw
  userland/capsule_input_probe/src/render/font.rs     ASCII to bitmap-row mapping
  userland/capsule_input_probe/src/render/font_table.rs  the built-in letter and digit bitmaps
  userland/capsule_input_probe/src/setup/fill.rs      the initial background fill
  userland/capsule_input_probe/src/state.rs           the Context surface and history buffer
```

Every reference above is verified against those trees.

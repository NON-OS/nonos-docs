# Debugging the toolkit

The toolkit is two things, and each fails in its own place. Library problems show up inside the capsule
that links it, not in a separate process, because a library call is an in-process function call with no
syscall or IPC. Service problems show up as wire replies from endpoint 4610. This page covers both. For
the module layout see the [README](README.md), the [library](library.md), and the [service](service.md)
pages in this folder.

## Library failure modes

Because the library runs in the linking capsule's address space, its bugs are that capsule's bugs, bounded
by the same page tables and the same capability token.

- Text misplaced or clipped. The usual cause is a stride passed in bytes instead of pixels: every drawing
  helper takes `stride` in pixels (`src/font/render/draw_glyph.rs:19`), and the service's `paint` derives
  it as `desc.stride / 4` (`src/component_dispatch/paint/paint.rs:35`). A linking capsule that hands a
  byte stride draws every row at four times the pitch.
- A glyph renders as the boxed unknown character. The byte fell through `glyph_for_ascii` to
  `GLYPH_UNKNOWN` (`src/font/glyph.rs:80`). The font is 8x8 ASCII plus a few icon bytes
  (`src/font/glyph.rs:56`), so anything outside that range is expected to box; this is not a bug.
- An image decode returns `OutputTooSmall`. The `out: &mut [u32]` was not sized to the decoded
  `width * height`. The decoders never grow the caller's buffer; they return a `DecodeError`
  (`src/image/types.rs:1`) rather than write past it. `BadMagic` and `Truncated` mean the input bytes are
  not the format claimed or end early.
- QR renders nothing. `render_matrix_argb8888` returns `false` if the matrix is shorter than `size * size`
  (`src/qr/render.rs:14`) or the target buffer is smaller than `size * scale` in either axis
  (`src/qr/render.rs:19`). A `false` return means the caller should widen the buffer or fix the matrix, not
  that pixels were dropped.

The structural guarantee is that a wrong stride or coordinate loses or misplaces pixels inside the
caller's own framebuffer and never corrupts memory outside it: every helper bounds its writes against
`buf.len()` with saturating arithmetic (`src/font/render/draw_glyph.rs:41`, `src/components/button.rs:36`,
`src/qr/render.rs:34`).

## The service spawn marker

The service is spawned as its own capsule. A successful spawn logs a line of the form:

```
  [SPAWN] name=toolkit pid=0x... caps=0x19 entry=0x...
```

`caps=0x19` confirms it was admitted with exactly `CoreExec | IPC | Memory` (`src/capabilities/types/defs.rs`,
`:59`, `:60`) and, by the analysis on the [service](service.md) page, that its `COMPONENT_RENDER` will
always answer `E_SURFACE` because it lacks `GraphicsSurfaceMap` (`src/capabilities/types/defs.rs`).

## Service wire signatures

A client reaches the service by looking up endpoint 4610 and sending an `NOTK` frame
(`src/protocol/header.rs:17`). Inside the receive loop, a non-positive length or a zero sender pid is
dropped silently (`src/server/runner.rs:38`), and a frame whose magic is not `0x4E4F544B` fails `decode`
and is dropped (`src/server/runner.rs:42`, `src/protocol/header.rs:33`). If `mk_ipc_recv_from` returns
`ENOTSUP` (`-95`), the runner exits with code 95 (`src/server/runner.rs:35`), which is the signature of a
kernel that does not support the receive syscall.

The per-op failure replies are:

- `E_BAD_OP` for an unknown op (`src/server/dispatch.rs:32`) or a `THEME_GET` reply buffer smaller than 24
  bytes (`src/server/dispatch.rs:37`).
- `E_SHORT` for a truncated `THEME_APPLY` (under 20 bytes, `src/theme/apply/apply.rs:22`) or a
  `COMPONENT_RENDER` payload shorter than the 28-byte header or with a label past its end
  (`src/component_dispatch/render/render.rs:24`, `:42`).
- `E_INVAL` for a `COMPONENT_RENDER` with a zero width, height, or handle
  (`src/component_dispatch/render/render.rs:37`).
- `E_SURFACE` when the surface attach fails, which under mask `0x19` is always
  (`src/component_dispatch/render/render.rs:46`; see the [service](service.md) page).

Because the theme is one global (`src/theme/store/state.rs:18`), a palette that looks wrong across every
app at once points at a stray `THEME_APPLY`, which the service does not authenticate, not at one client.
The shared animation counter (`src/animation/store/advance.rs:20`) is likewise global, so a counter that
jumps is a concurrent ticker, by design.

## Source map

```
  userland/toolkit/src/font/render/draw_glyph.rs      pixel-stride and buf.len() bounds
  userland/toolkit/src/font/glyph.rs                  GLYPH_UNKNOWN fallback
  userland/toolkit/src/image/types.rs                 DecodeError variants
  userland/toolkit/src/qr/render.rs                   the false-return guards
  userland/toolkit/src/server/runner.rs               receive loop, ENOTSUP exit, silent drops
  userland/toolkit/src/server/dispatch.rs             per-op status routing
  userland/toolkit/src/component_dispatch/render/render.rs   render parse and E_SURFACE
  userland/toolkit/src/theme/store/state.rs           the global palette atoms
  src/capabilities/types/defs.rs                           the 0x19 bit decomposition and GraphicsSurfaceMap
```

Every reference above is verified against those trees.

# toolkit

The toolkit is the shared GUI library that every NONOS GUI capsule links against. It is one crate,
`nonos_toolkit` (`userland/toolkit/Cargo.toml`), that ships two things from the same source tree: a
`#![no_std]` library (`[lib] name = "nonos_toolkit"`, `src/lib.rs`) that GUI capsules compile into their
own binary, and a small `toolkit` service binary (`[[bin]] name = "toolkit"`, `src/main.rs`) that answers
a few operations over IPC. The library is the real product. It gives a capsule a font renderer, a design
vocabulary, window decorations, a set of widget helpers, image decoders, and a QR renderer, all of which
run inside the calling capsule's address space and paint into a framebuffer the capsule already owns. The
crate is `no_std` and pulls in only `alloc`, `nonos_userland_libc`, and `spin` (`Cargo.toml`).

## What links it

The library is linked by the capsules that draw. Their `Cargo.toml` files carry
`nonos_toolkit = { path = "../toolkit" }`: `app_skeleton`, `capsule_desktop_shell`, `capsule_boot_splash`,
`capsule_image_codec`, `capsule_setup_wizard`, and `capsule_wallpaper` all declare it. In their source
they reach in by path, not by IPC: `app_skeleton` uses `nonos_toolkit::font::render::draw_text`
(`userland/app_skeleton/src/paint/text.rs:17`) and `nonos_toolkit::decorations::{hit_test, DecorationHit}`
(`userland/app_skeleton/src/runner/decorations.rs:19`); `capsule_image_codec` uses
`nonos_toolkit::image::{bmp, jpeg, lz4_raw, png::decoder, types::DecodeError}`
(`userland/capsule_image_codec/src/server/handlers/decode.rs:18`). That is the primary way the toolkit is
used: as ordinary function calls, in-process, with no capability crossing and no service round trip.

## Module layout

`src/lib.rs:21` declares the modules and re-exports them:

```
  animation           easing, timing, transitions, an animation store, tick()
  component_dispatch   the service-side COMPONENT_RENDER path (kind, paint, render)
  components           widget helpers: button, label, checkbox, radio, toggle, ...
  decorations          window chrome: titlebar, close/min/max buttons, borders, hit test
  design               color, spacing, border/radius, shadow, typography
  font                 an 8x8 bitmap atlas and text/glyph drawing
  image                bmp, png, jpeg, and a raw lz4 decoder to ARGB8888
  protocol             the NOTK IPC header, ops, and error codes for the service
  qr                   QR matrix generation (ecc, format, mask, place) and a renderer
  server               the port-4610 receive loop for the service binary
  theme                a global palette snapshot with a revision counter
```

## The drawing surface interface

Every drawing helper in the library sits on the same flat interface: an ARGB8888 pixel buffer the caller
already has, plus its geometry. The signature is stable across the crate, seen in `draw_glyph`
(`src/font/render/draw_glyph.rs:18`), `draw_text` (`src/font/render/draw_text.rs:20`), `render_button`
(`src/components/button.rs:45`), and `render_matrix_argb8888` (`src/qr/render.rs:1`):

```
  buf: &mut [u32]   the caller's framebuffer, one ARGB8888 pixel per u32
  stride: usize     pixels per row (not bytes)
  w, h: u32         clip bounds; nothing is written past them
  x, y: u32         where to draw
  ...               glyph, color, label, style, or matrix
```

The helpers never allocate a surface and never present. They compute a pixel index
`py * stride + px`, bound it against `buf.len()`, and write a `u32`. `draw_glyph`
(`src/font/render/draw_glyph.rs:30`) skips any pixel that lands outside `w`/`h` or past the slice; the
`fill_rect` inside `render_button` (`src/components/button.rs:16`) clamps its rectangle to `w`/`h` and to
the slice with `saturating_mul`/`saturating_add`. So a wrong stride or an out-of-range coordinate loses
pixels rather than corrupting memory outside the buffer the caller handed in. The capsule that owns the
surface is the one that maps it, paints into it with these helpers, and presents it. The toolkit library
just fills the bytes.

## The font

`font` is a fixed 8x8 bitmap font, not a glyph shaper. `FontAtlas` (`src/font/atlas.rs`) defaults to
`glyph_width = 8`, `glyph_height = 8`, `letter_spacing = 1`, and `text_width` returns
`len * 8 + (len - 1) * spacing`. `GlyphBitmap` (`src/font/glyph.rs:1`) is `width`, `height`, and
`rows: [u8; 8]`, one bit per pixel, MSB first. `glyph_for_ascii` (`src/font/glyph.rs:52`) maps a byte to a
glyph: space, the digits `0`-`9`, ASCII upper (`src/font/upper.rs`) and lower (`src/font/lower.rs`) cases,
punctuation (`src/font/punct.rs`), plus a few control bytes reused as icons (`0xD8` is a slashed O, `0x10`
a chevron, `0x11` a check, `0x12` a cross), and everything else falls back to `GLYPH_UNKNOWN`. `draw_text`
(`src/font/render/draw_text.rs`) walks the bytes, drawing each glyph and advancing the pen by
`glyph_width + letter_spacing`. `font::render` also exports `draw_glyph_scaled` and `draw_text_scaled`
(`src/font/render/mod.rs:21`) for integer-scaled text.

## The design vocabulary

`design` is plain value types with no drawing of their own. `Argb` (`src/design/color.rs:1`) wraps a
`u32` with `from_channels`, `with_alpha`, `alpha`, `as_u32`, and the constants `BLACK`, `WHITE`,
`TRANSPARENT`. `Palette` (`src/design/color.rs:26`) is `background`/`foreground`/`accent`/`danger` with a
dark default. `TextStyle` (`src/design/typography.rs:8`) carries `px`, `FontWeight`
(`Regular`/`Medium`/`Bold`), `letter_spacing`, `line_height`, with `caption`/`body`/`title`/`headline`
presets. `SpacingScale` (`src/design/spacing.rs:19`) is a `4/8/12/16/24` step scale and `Insets`
(`src/design/spacing.rs:1`) is four-sided padding. `Border` and `Radius` (`src/design/border.rs`) describe
a border width, color, and per-corner radius. `Shadow` (`src/design/shadow.rs`) is offset, blur, spread,
and color with `none`/`sm`/`md` presets. These are the tokens a capsule reads to stay visually consistent;
nothing in `design` writes a pixel.

## The widgets

`components` (`src/components/mod.rs`) is a set of small, mostly stateless helpers. They fall into two
shapes. Some render directly into the surface buffer: `render_button` (`src/components/button.rs:45`) with
`ButtonStyle { bg, fg }`, `render_label` (`src/components/label.rs:15`) with `LabelStyle { color }`,
`render_slider` (`src/components/slider.rs`), `render_input` (`src/components/input.rs`), and
`render_list_item` (`src/components/list.rs`). Others are pure state or style logic the capsule uses to
decide colors and positions: `checkbox_color` (`src/components/checkbox.rs`), `radio_color`
(`src/components/radio.rs`), `toggle_track` (`src/components/toggle.rs`), `progress_pct`
(`src/components/progress.rs`), `gradient_color` (`src/components/colorpicker.rs`), `is_valid_date` over
`CalendarDate` (`src/components/datepicker.rs`), `first_enabled` over `MenuItem`
(`src/components/menu.rs`), and the small state types `ScrollState` (`src/components/scroll.rs`),
`TabBarState` (`src/components/tabbar.rs`), and `StatusFlags` (`src/components/statusbar.rs`). `dropdown`
(`src/components/dropdown/`), `card`, `badge`, `tooltip`, and `glass_panel` round out the set. Each style
struct derives `Default`, so a capsule can take the defaults or override a field.

## Window decorations

`decorations` (`src/decorations/mod.rs`) draws window chrome and answers hit tests, so every window frame
looks the same. It exports `draw_titlebar` (`src/decorations/titlebar.rs`), `draw_border`, the three
window buttons `draw_close_button`/`draw_minimize_button`/`draw_maximize_button` with their `*_rect`
geometry helpers, and `hit_test` returning `DecorationHit` (`src/decorations/hit_test.rs:21`), which is
`None`, `Titlebar`, `CloseButton`, `MinimizeButton`, or `MaximizeButton`. The chrome metrics are fixed
constants (`src/decorations/metrics.rs:17`): `TITLEBAR_HEIGHT = 26`, `TITLEBAR_PADDING = 10`,
`TITLE_TEXT_Y = 9`, `CLOSE_BUTTON_SIZE = 18`, `BUTTON_GAP = 6`, `BORDER_PX = 1`. `hit_test` reads those
same rects, so a click is classified against exactly the chrome that was drawn. `app_skeleton` uses this
directly for its own window frame (`userland/app_skeleton/src/runner/decorations.rs:19`).

## Image decoders

`image` (`src/image/mod.rs`) decodes to ARGB8888 in the caller's buffer. Every entry point shares the
signature `(input: &[u8], out: &mut [u32]) -> Result<ImageSize, DecodeError>`: `decode_bmp_argb8888`
(`src/image/bmp.rs:13`), `decode_png_argb8888` (`src/image/png/decoder.rs:11`), `decode_jpeg_argb8888`
(`src/image/jpeg/decode/decode_jpeg_argb8888.rs:30`), and `decode_lz4_raw_argb8888`
(`src/image/lz4_raw.rs:3`). `ImageSize` (`src/image/types.rs:11`) is `width`/`height` and refuses zero
dimensions; `DecodeError` (`src/image/types.rs:2`) is `BadMagic`, `Unsupported`, `BadDimensions`,
`OutputTooSmall`, or `Truncated`. The PNG path is a full inflate (`src/image/png/inflate/`) plus scanline
defilter (`src/image/png/scanline.rs`); the JPEG path is a baseline decoder with its own Huffman tables,
dequant, IDCT, and YCbCr conversion (`src/image/jpeg/`). `capsule_image_codec` is built on exactly these
(`userland/capsule_image_codec/src/server/handlers/decode.rs:18`).

## QR rendering

`qr` (`src/qr/mod.rs`) builds a QR matrix (`ecc`, `format`, `mask`, `place`) and paints it.
`render_matrix_argb8888` (`src/qr/render.rs:1`) takes the matrix, its `size`, an integer `scale`, an `on`
and `off` color, and the usual `buf`/`stride`/`w`/`h`, and returns `false` if the matrix is short or the
buffer is too small for `size * scale`, otherwise it blits scaled modules.

## The service binary

The same crate also builds a `toolkit` service. It exists to hold a global theme and an animation counter
that any capsule can read over IPC, and to expose the component paint path as an RPC. `main.rs`
(`src/main.rs:26`) initializes the heap (tolerating an already-initialized heap) and calls
`server::runner::run` (`src/server/runner.rs:29`), which loops: receive on the endpoint, decode the
header, dispatch, encode the reply. The wire frame is `NOTK` (magic `0x4E4F544B`,
`src/protocol/header.rs:17`) with a 16-byte header carrying the op, a status, a request id, and a payload
length. The endpoint is `4610` (`src/protocol/ops.rs:17`), and `IPC_PAYLOAD_MAX` is 256 bytes.

Five operations (`src/protocol/ops.rs:19`):

```
  HEALTHCHECK = 0x0000   THEME_APPLY = 0x0001   ANIMATION_TICK = 0x0002
  COMPONENT_RENDER = 0x0003   THEME_GET = 0x0004
```

`dispatch` (`src/server/dispatch.rs:25`) routes each:

- `HEALTHCHECK` replies `STATUS_OK` with an empty body.
- `THEME_APPLY` calls `theme::apply` (`src/theme/apply/apply.rs:21`), which requires at least 20 bytes
  (five little-endian ARGB colors) or returns `E_SHORT`, then replaces the global palette
  (`src/theme/store/replace.rs`). The stored revision is bumped by the store, not taken from the payload.
- `THEME_GET` returns a 24-byte snapshot (`src/server/dispatch.rs:36`): background, surface, accent, text,
  border, and a `u32` revision, all little-endian. A reply buffer shorter than `THEME_PAYLOAD_LEN` (24)
  comes back `E_BAD_OP`.
- `ANIMATION_TICK` calls `animation::tick` (`src/animation/tick.rs:21`), reads an optional 8-byte delta,
  advances the global `TICK` counter (`src/animation/store/advance.rs:20`; a zero delta advances by one),
  and returns the new counter as 8 little-endian bytes.
- `COMPONENT_RENDER` calls `component_dispatch::render` (`src/component_dispatch/render/render.rs:23`).

The theme store is atomics with a default dark palette (`src/theme/store/state.rs`), and `snapshot`
(`src/theme/store/snapshot.rs`) reads them with `Acquire`. The animation `advance` uses `AcqRel` on a
shared counter, so concurrent ticks from different callers race on that one counter by design.

## COMPONENT_RENDER and the surface it cannot map

`COMPONENT_RENDER` is where the service tries to draw, and it is worth being exact about it because the
capability model decides the outcome. `render` (`src/component_dispatch/render/render.rs:23`) parses a
28-byte header (`HEADER_LEN`, `src/component_dispatch/render/constants.rs:16`): a `u64` surface handle,
then `x`/`y`/`w`/`h`, a `u16` kind, and a `u16` label length, followed by the label bytes. It rejects a
short payload (`E_SHORT`), a zero width/height/handle (`E_INVAL`), and a label that runs past the payload
(`E_SHORT`). It then calls `attached_surface(handle)` (`src/component_dispatch/render/attached_surface.rs:23`),
which caches one attachment and otherwise calls `mk_surface_attach` from
`nonos_libc`. If that returns `<= 0` it yields `E_SURFACE`; on success it paints into the mapped surface
VA via `paint` (`src/component_dispatch/paint/paint.rs:26`), building the `buf` slice from the descriptor
`base_va`, `stride`, and `height` and drawing a panel, a themed button, or a label by
`ComponentKind::from_raw` (`src/component_dispatch/kind.rs:24`).

The point is that `mk_surface_attach` is gated. In the kernel, `MkSurfaceAttach` requires
`caps.can_surface_map()` (`src/syscall/contract/cap_table/mk.rs:75`), which is
`grants(Capability::GraphicsSurfaceMap)` (`src/syscall/caps/checks/graphics.rs:29`), bit `8192`
(`src/capabilities/types.rs:69`). The toolkit capsule is admitted with `CAPSULE_REQUIRED_CAPS = 0x19`
(`userland/toolkit/Capsule.mk:11`), which decodes to `CoreExec | IPC | Memory`
(`1 | 8 | 16`, `src/capabilities/types.rs`) and holds no graphics bit. So the service, as configured,
cannot attach any surface: `mk_surface_attach` fails the capability check, `attached_surface` returns
`None`, and `COMPONENT_RENDER` always answers `E_SURFACE`. The paint path is present in the code but
unreachable from the service's own token. Capsules that actually draw do so through the library, in their
own address space, using surfaces they own; they do not go through the service's `COMPONENT_RENDER`.

## Security analysis

The library is not a trust boundary. When a GUI capsule links `nonos_toolkit`, the font renderer, the
decorations, the widget helpers, and the image decoders all run in that capsule's own address space with
that capsule's own capabilities. There is no syscall, no IPC, and no privilege change on a `draw_text` or a
`decode_png_argb8888` call. A bug in the toolkit library is a bug in the linking capsule, bounded by the
same page tables and the same capability token the capsule already has. This is the important structural
fact: the toolkit does not sit between a capsule and the kernel, so it cannot widen what a capsule can do.

- **Buffer-bounded drawing.** Every drawing helper clips to the caller-supplied `w`/`h` and bounds its
  writes against `buf.len()` with saturating arithmetic (`src/font/render/draw_glyph.rs:37`,
  `src/components/button.rs:27`, `src/qr/render.rs:14`). The worst a wrong stride or coordinate does is
  drop or misplace pixels inside the caller's own framebuffer. The helpers hold no pointer the caller did
  not give them.
- **Untrusted image input is the real attack surface.** `capsule_image_codec` and `capsule_wallpaper`
  feed attacker-controllable bytes into the PNG and JPEG decoders. Those decoders are the part of the
  library that parses hostile data, and their `DecodeError` variants (`BadMagic`, `Truncated`,
  `OutputTooSmall`, `BadDimensions`, `Unsupported`, `src/image/types.rs:2`) are the guard. Because the
  decoders run in the codec capsule's address space, a decoder flaw is contained to that capsule, not to
  every GUI app that draws.
- **The service is a leaf that cannot paint.** The `toolkit` service (`src/server/dispatch.rs:25`) calls
  no other service; every op is answered from the global theme atoms and the animation counter. Its
  `COMPONENT_RENDER` op would attach and paint a surface, but with caps `0x19` it lacks
  `GraphicsSurfaceMap` and always returns `E_SURFACE` (see the section above), so a compromise of the
  service cannot put pixels on any surface. It also holds no `GraphicsSurfaceCreate` or `GraphicsPresent`
  (`src/capabilities/types.rs:68`, `:70`), so it can neither create a surface nor present one.
- **Honest boundary: the service has no caller authentication.** Any capsule that can reach endpoint 4610
  can `THEME_APPLY` and change the palette every reader gets from `THEME_GET`, and any caller can tick the
  shared animation counter. `THEME_APPLY` does validate length (at least 20 bytes,
  `src/theme/apply/apply.rs:22`) but does not check who is calling. The model treats the theme as
  cosmetic: a stray theme is ugly, not a boundary crossing, because the service cannot draw it anywhere
  itself. The shared animation counter means concurrent ticks race by design.

## Debugging

Most toolkit problems are library problems, so they show up in the capsule that links it, not in a
separate process. If text is misplaced or clipped, the usual cause is a stride passed in bytes instead of
pixels: the helpers take `stride` in pixels (`src/font/render/draw_glyph.rs:19`), and `paint` derives it
as `desc.stride / 4` (`src/component_dispatch/paint/paint.rs:35`). If a glyph renders as the boxed unknown
character, the byte fell through `glyph_for_ascii` to `GLYPH_UNKNOWN` (`src/font/glyph.rs:80`); the font is
8x8 ASCII plus a few icon bytes, so anything outside that range is expected to box. An image that returns
`OutputTooSmall` means the `out: &mut [u32]` was not sized to the decoded `width * height`.

For the service, it registers on endpoint 4610 and receives there (`src/server/runner.rs:34`). A client
reaches it by looking the endpoint up and sending an `NOTK` frame; a zero pid or a header whose magic is
not `0x4E4F544B` is dropped silently in the loop (`src/server/runner.rs:38`, `:42`). If
`mk_ipc_recv_from` returns `ENOTSUP` (`-95`), the runner exits with code 95 (`src/server/runner.rs:35`),
which is the signature of a kernel that does not support the receive syscall. The spawn marker is:

```
  [SPAWN] name=toolkit pid=0x... caps=0x19 entry=0x...
```

`caps=0x19` confirms it was admitted with exactly `CoreExec | IPC | Memory` and, by the analysis above,
that its `COMPONENT_RENDER` will always answer `E_SURFACE` because it lacks `GraphicsSurfaceMap`. The wire
failure signatures are `E_BAD_OP` for an unknown op or a too-small `THEME_GET` reply
(`src/server/dispatch.rs:32`, `:37`), `E_SHORT` for a truncated `THEME_APPLY` or `COMPONENT_RENDER`
header, `E_INVAL` for a zero-sized or zero-handle render, and `E_SURFACE` when the surface attach fails
(`src/protocol/errno.rs`). Because the theme is one global, a palette that looks wrong across every app at
once points at a stray `THEME_APPLY`, not at one client.

## Source map

```
  userland/toolkit/Cargo.toml                       lib nonos_toolkit + bin toolkit, deps
  userland/toolkit/src/lib.rs                        module tree and re-exports
  userland/toolkit/src/font/                          8x8 atlas, glyph tables, text/glyph drawing
  userland/toolkit/src/design/                        color, spacing, border, shadow, typography tokens
  userland/toolkit/src/components/                     widget render and state helpers
  userland/toolkit/src/decorations/                    titlebar, window buttons, borders, hit test
  userland/toolkit/src/image/                          bmp, png, jpeg, lz4 decoders to ARGB8888
  userland/toolkit/src/qr/                             QR matrix build and render
  userland/toolkit/src/animation/                      easing, timing, the shared tick counter
  userland/toolkit/src/theme/                          global palette store, apply, snapshot
  userland/toolkit/src/protocol/                       NOTK header, ops, error codes
  userland/toolkit/src/server/                         port-4610 receive loop and dispatch
  userland/toolkit/src/component_dispatch/             COMPONENT_RENDER: parse, attach, paint
  userland/toolkit/Capsule.mk                          CAPSULE_REQUIRED_CAPS = 0x19, endpoint 4610
  userland/libc/src/surface_registry/                  mk_surface_attach, SurfaceDescriptor
  src/syscall/contract/cap_table/mk.rs                 MkSurfaceAttach needs can_surface_map()
  src/syscall/caps/checks/graphics.rs                  can_surface_map = GraphicsSurfaceMap
  src/capabilities/types.rs                            capability bit values (0x19 = CoreExec|IPC|Memory)
```

Every reference above is verified against those trees.

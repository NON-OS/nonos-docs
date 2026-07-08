# capsule_image_codec

The image codec decodes compressed images into raw ARGB pixels for the rest of the desktop. It is a
stateless service: hand it PNG, BMP, JPEG, or raw-LZ4 bytes and it returns a decoded surface. Service
`image_codec` on port 4412, capability mask `0x1819`. The source is `userland/capsule_image_codec/`.

## The server loop

`main.rs:28` initializes the heap and runs the loop (`src/server/runner.rs:28`) with a large (128 KiB)
payload buffer to hold decoded output: receive, parse, dispatch, reply. The frame is `GMIN` (magic
`0x474D494E`), version 1.

## The operations

Five operations (`src/protocol/ops.rs:17`):

```
  HEALTHCHECK=1  DECODE_PNG=2  DECODE_BMP=3  DECODE_LZ4_RAW=4  DECODE_JPEG=5
```

The decode handler (`src/server/handlers/decode.rs:25`) allocates a pixel buffer and dispatches to the
format decoder, then registers the decoded pixels as an ARGB surface and returns the surface handle,
dimensions, stride, and byte length:

```
  handle(req, body):
      pixels = [0u32; MAX_PIXELS]
      decoded = match op:
          DECODE_PNG      -> decode_png_argb8888(body, pixels)
          DECODE_BMP      -> decode_bmp_argb8888(body, pixels)
          DECODE_JPEG     -> decode_jpeg_argb8888(body, pixels)
          DECODE_LZ4_RAW  -> decode_lz4_raw_argb8888(body, pixels)
      (handle, stride, len) = register_argb_surface(pixels[..decoded])
      -> handle, width, height, stride, byte_len
```

## Real decoders

The decoders are not hand-rolled placeholders: they come from the `nonos_toolkit::image` crate,
`png::decoder`, `bmp`, `jpeg`, and `lz4_raw`, so the codec runs real format parsers over
attacker-influenced input (every image is untrusted bytes), which is exactly why it is isolated in its
own capsule. A decode failure returns an error rather than a partial surface.

## Honest scope

The service is stateless and holds no session between requests; the returned surface lives in the shared
surface registry. The one stated limit is the pixel budget: the decode buffer caps the output size (a
16K-pixel limit), so a very large image is not decoded in one call, and there is no streaming decode.

## Source

```
  userland/capsule_image_codec/src/server/runner.rs     the loop
  userland/capsule_image_codec/src/server/handlers/decode.rs   the format dispatch + surface register
  (decoders) nonos_toolkit::image::{png, bmp, jpeg, lz4_raw}
```

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
surface registry. The one stated limit is the pixel budget: the decode buffer is `MAX_PIXELS = 16384`
32-bit pixels (`src/server/handlers/decode.rs:23`), so a very large image is not decoded in one call, and
there is no streaming decode. The IPC payload cap is 128 KiB (`IPC_PAYLOAD_MAX = 131072`,
`src/protocol/limits.rs:17`), which bounds the compressed input a single request can carry.

## Security analysis

The mask is `0x1819` (`CAPSULE_REQUIRED_CAPS` in `userland/capsule_image_codec/Capsule.mk`), decoding to
`CoreExec | IPC | Memory | GraphicsDisplayQuery | GraphicsSurfaceCreate` against
`src/capabilities/types.rs`. It has `GraphicsSurfaceCreate` because a decode ends by registering the
decoded pixels as an ARGB surface and returning the handle (`src/server/handlers/surface.rs:48`,
`mk_surface_register`), so the caller gets a surface it can hand to the compositor rather than a raw copy
of the pixels back over IPC. It does not hold `GraphicsSurfaceMap` or `GraphicsPresent`: the codec creates
surfaces, it does not map arbitrary ones or paint to the screen.

The reason this capsule exists at all is isolation, and the mask is what makes the isolation worth having.

- **Untrusted input, contained blast radius.** Every image is attacker-influenced bytes, and the decoders
  are real format parsers (`nonos_toolkit::image::{png, bmp, jpeg, lz4_raw}`) running over them, which is
  exactly the class of code that has parser bugs. The mask has no `FileSystem`, no `Network`, no
  `Hardware`, and no `Crypto`, so a decoder that is subverted by a malformed image is trapped in a capsule
  that can create a surface and reply, and nothing else. It cannot read a file, open a socket, or reach a
  device off the back of a parser exploit.
- **Bounded output.** The fixed `MAX_PIXELS` buffer means a crafted image cannot make the codec allocate an
  unbounded surface, and a decode that would overflow the budget fails rather than truncating into a partial
  surface (`src/server/handlers/decode.rs`).
- **Stateless.** No session is carried between requests, so one caller's image cannot influence another's
  decode; the only shared thing produced is the surface, which is owned by the surface registry once
  registered.

## Debugging

The codec registers as `image_codec` on port 4412 and is reached by `mk_service_lookup("image_codec")`.
The kernel spawn marker is:

```
  [SPAWN] name=image_codec pid=0x... caps=0x1819 entry=0x...
```

`caps=0x1819` confirms it was admitted with `CoreExec | IPC | Memory | GraphicsDisplayQuery |
GraphicsSurfaceCreate`; if a build ever changed the requested caps, that number is where it shows.

The failure signatures are on the wire. A decode that fails (a truncated or malformed image, or one that
exceeds the pixel budget) returns an error status through `fail` rather than a surface handle
(`src/server/handlers/decode.rs:36`), so a caller that gets a nonzero status and no handle is looking at a
bad image, not a dead codec. An op outside 1..5 is rejected. A request whose compressed body exceeds the
128 KiB payload cap is refused before decode. Because the service is stateless, a codec that decodes one
image and fails the next is behaving correctly on two different inputs, not degrading.

## Source map

```
  userland/capsule_image_codec/src/server/runner.rs            the 128 KiB-buffer loop
  userland/capsule_image_codec/src/server/handlers/decode.rs   format dispatch, MAX_PIXELS, fail path
  userland/capsule_image_codec/src/server/handlers/surface.rs  register the decoded ARGB surface
  userland/capsule_image_codec/src/protocol/{ops.rs, limits.rs} the GMIN ops and IPC_PAYLOAD_MAX
  userland/capsule_image_codec/Capsule.mk                      CAPSULE_REQUIRED_CAPS = 0x1819, endpoint 4412
  (decoders) nonos_toolkit::image::{png, bmp, jpeg, lz4_raw}
```

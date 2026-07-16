# capsule_image_codec (full reference)

`capsule_image_codec` is a decode service. It takes encoded image bytes over IPC, runs a real format
parser over them, and hands back a shared ARGB8888 surface handle that another capsule can present. It
holds no session and it renders nothing itself; it turns compressed bytes into pixels and gets out of
the way. The kernel spawns it under service handle `image_codec` on service port 4412 with a reply port
on 4413, and its capability mask is `0x1819` (`userland/capsule_image_codec/Capsule.mk:8`,
`Capsule.mk:9`, `Capsule.mk:11`). The source is `userland/capsule_image_codec/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The codec is a `no_std`/`no_main` capsule whose `_start` initializes the heap and enters the server loop
(`src/main.rs:28`). The loop receives one framed request at a time, parses a fixed header, dispatches on
the opcode, and replies (`src/server/runner.rs:28`). A decode request carries the compressed image in the
payload; the handler allocates a fixed pixel buffer, runs the matching decoder over the untrusted bytes,
registers the decoded pixels as an ARGB8888 surface, and returns the surface handle plus its geometry
(`src/server/handlers/decode.rs:25`). A failed decode returns a typed error status and no handle, never a
partial surface (`src/server/handlers/decode.rs:34`, `decode.rs:47`).

The decoders are the real thing, not placeholders. They come from the shared `nonos_toolkit::image`
crate: `png::decoder::decode_png_argb8888`, `bmp::decode_bmp_argb8888`, `jpeg::decode_jpeg_argb8888`, and
`lz4_raw::decode_lz4_raw_argb8888` (`src/server/handlers/decode.rs:18`, `decode.rs:28`). Every image the
codec sees is attacker-influenced input, and running a full PNG/JPEG parser over such input is exactly the
class of code that has parser bugs, which is the entire reason this work runs in its own capsule with a
narrow mask rather than inside the compositor or a caller.

The service is spawned as part of the desktop services fleet at boot (`spawn_image_codec` in
`src/userspace/init/spawn_plan/desktop_services.rs:43`), and it is compiled into the
`microkernel-desktop-gui` profile through the `nonos-capsule-image-codec` feature (`Cargo.toml:84`,
`Cargo.toml:469`). At the time of writing no in-tree capsule calls the service over IPC; the wallpaper
capsule, for instance, decodes its PNG in-process with the same toolkit decoder rather than through this
service (`userland/capsule_wallpaper/src/decode_client/wire.rs:21`). The service exists as the isolated
decode boundary for callers that want the parser off their own stack.

## Identity

Everything the kernel and the service registry need to name and reach the codec comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `image-codec` | `Capsule.mk:1` |
| Service handle | `image_codec` | `Capsule.mk:2`, `src/userspace/capsule_image_codec/spawn.rs:30` |
| Namespace | `systems.nonos.image_codec` | `Capsule.mk:7` |
| Service endpoint | `service:4412:image_codec` | `Capsule.mk:8`, `spawn.rs:31` |
| Reply endpoint | `reply:4413:endpoint.image_codec.reply` | `Capsule.mk:9`, `spawn.rs:32`, `spawn.rs:33` |
| Capability mask | `0x1819` | `Capsule.mk:11` |
| Binary name | `image_codec` | `Capsule.mk:5` |
| Kernel mirror | `src/userspace/capsule_image_codec` | `Capsule.mk:12` |

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

That is CoreExec, IPC, Memory, GraphicsDisplayQuery, and GraphicsSurfaceCreate, and nothing else. The bit
for `Debug` is `0x0100` (`types.rs:64`) and it is not set, so the comment in `Capsule.mk:10` that lists
`Debug` is stale relative to the actual value on `Capsule.mk:11`; the mask is `0x1819`, not `0x1919`.
There is no `Network` bit (4), no `FileSystem` bit (64), no `Crypto` (32), no `Hardware` (128), and no
driver, MMIO, IRQ, DMA, or PIO capability. There is also no `GraphicsSurfaceMap` (8192) and no
`GraphicsPresent` (16384): the codec creates a surface and shares its handle, but it does not map
arbitrary surfaces and it does not paint to the screen.

The mask on the running process is decided by the signed manifest, not by the number passed at the spawn
site. The kernel spawn record requests `0x19` as its ceiling for optional caps
(`src/userspace/capsule_image_codec/spawn.rs:35`), but `spawn_verified` installs the caps from the
verified manifest, and `requested_caps` is only the upper bound for optional caps
(`src/kernel_core/process_spawn/capsule_spawn/runner/verified.rs:23`). The manifest is signed from
`Capsule.mk` with `--required-caps 0x1819` and `--optional-caps 0x0` (`nonos-mk/capsule.mk:230`,
`nonos-mk/capsule.mk:231`, default at `nonos-mk/capsule.mk:70`). The install computes
`required_caps | (optional_caps & granted_caps)`
(`src/security/capsule_manifest/verify/caps.rs:39`), which with all caps declared required and no optional
caps resolves to exactly `0x1819`. That is the number the `[SPAWN]` line reports below.

## Operation reference

The frame is `GMIN`, magic `0x474D494E` little-endian, version 1, with a 20-byte header
(`src/protocol/header.rs:17`, `header.rs:18`, `header.rs:19`). Five operations are defined
(`src/protocol/ops.rs:17`):

| Op | Opcode | Body | What it does | Handler |
|---|---|---|---|---|
| `OP_HEALTHCHECK` | `0x0001` | empty | reply with status 0, no payload | `ops.rs:17`, `runner.rs:52`, `handlers/health.rs:20` |
| `OP_DECODE_PNG` | `0x0002` | PNG bytes | decode a PNG to ARGB8888, register a surface | `ops.rs:18`, `runner.rs:53`, `handlers/decode.rs:28` |
| `OP_DECODE_BMP` | `0x0003` | BMP bytes | decode a BMP to ARGB8888, register a surface | `ops.rs:19`, `runner.rs:53`, `handlers/decode.rs:29` |
| `OP_DECODE_LZ4_RAW` | `0x0004` | `width(4) height(4) lz4-raw-argb` | decode raw LZ4-ARGB with a dimension prefix | `ops.rs:20`, `runner.rs:53`, `handlers/decode.rs:31` |
| `OP_DECODE_JPEG` | `0x0005` | JPEG bytes | decode a baseline JPEG to ARGB8888, register a surface | `ops.rs:21`, `runner.rs:53`, `handlers/decode.rs:30` |

The three container formats (PNG, BMP, JPEG) carry the image bytes directly in the payload, and the
decoder reads the dimensions out of the format's own header. `OP_DECODE_LZ4_RAW` is not a container
format: the first eight payload bytes are a little-endian `width` and `height`, and the remainder is raw
ARGB8888 already expanded (four bytes per pixel), which the handler forwards to the LZ4 decoder with the
parsed dimensions (`handlers/decode.rs:49`). The prefix length is `DECODE_LZ4_PREFIX_LEN = 8`
(`src/protocol/limits.rs:19`); a body shorter than that returns truncated.

The decode reply body is fixed at `DECODE_RESP_LEN = 32` bytes (`src/protocol/limits.rs:20`), written
after the 4-byte status word at offset `HDR_LEN + STATUS_LEN` (`handlers/decode.rs:37`):

```
  +0   u64  surface handle (from mk_surface_share)   decode.rs:38
  +8   u32  width                                    decode.rs:39
  +12  u32  height                                   decode.rs:40
  +16  u32  stride (width * 4 bytes)                 decode.rs:41
  +20  u32  format tag = 1 (ARGB8888)                decode.rs:42
  +24  u64  byte length (stride * height)            decode.rs:43
```

Size limits:

- The decode buffer is `MAX_PIXELS = 16384` 32-bit pixels (`handlers/decode.rs:23`, `decode.rs:26`), so an
  image whose pixel count exceeds that budget fails rather than truncating. Each toolkit decoder checks
  its own output against the buffer length before it writes (for example `lz4_raw.rs:11`,
  `image/types.rs:18` on zero dimensions).
- The IPC payload cap is `IPC_PAYLOAD_MAX = 131072` bytes (`src/protocol/limits.rs:17`). The receive
  buffer is `HDR_LEN + IPC_PAYLOAD_MAX` (`runner.rs:29`), so a single request's compressed input is
  bounded by 128 KiB.

Error codes are typed `i32` values returned in the status word (`src/protocol/errno.rs`):

| Error | Value | Meaning | Where raised |
|---|---|---|---|
| `E_INVAL` | `-22` | bad body, or a decoder `BadMagic` | `errno.rs:17`, `runner.rs:55`, `decode.rs:57` |
| `E_NOMEM` | `-12` | surface allocation or registration failed | `errno.rs:22`, `handlers/surface.rs:34`, `surface.rs:51` |
| `E_BAD_OP` | `-38` | unknown opcode with an empty body | `errno.rs:18`, `runner.rs:54` |
| `E_BAD_MAGIC` | `-74` | frame magic is not `GMIN` | `errno.rs:20`, `protocol/decode.rs:33` |
| `E_BAD_LEN` | `-90` | short header, declared length mismatch, or a decoder length error | `errno.rs:19`, `protocol/decode.rs:26`, `decode.rs:63`, `handlers/decode.rs:57` |
| `E_BAD_VERSION` | `-93` | header version is not 1 | `errno.rs:21`, `protocol/decode.rs:52` |
| `E_UNSUPPORTED` | `-95` | the format is recognised but a feature is not supported | `errno.rs:23`, `handlers/decode.rs:57` |

The toolkit `DecodeError` variants map onto those codes in one place
(`map_decode_error`, `handlers/decode.rs:56`): `BadMagic` to `E_INVAL`, `Unsupported` to `E_UNSUPPORTED`,
and `BadDimensions`, `OutputTooSmall`, and `Truncated` all to `E_BAD_LEN`
(`nonos_toolkit::image::types::DecodeError`, `userland/toolkit/src/image/types.rs:1`).

## Architecture and lifecycle

The capsule has two top-level modules: `protocol`, the wire format, and `server`, the loop and handlers
(`src/main.rs:22`, `main.rs:23`).

The `protocol` module owns the frame. `header.rs` defines the `GMIN` magic, version, 20-byte header, and
the `Request` struct (op, flags, request id). `decode.rs` parses a received buffer, validating magic,
version, header length, and that the declared payload length exactly matches the buffer tail, and returns
the `Request` and a body slice (`protocol/decode.rs:19`). `encode.rs` writes the response header and the
status word (`protocol/encode.rs:19`, `encode.rs:29`). `errno.rs`, `limits.rs`, and `ops.rs` are the
constants above, and `mod.rs` re-exports the surface (`protocol/mod.rs:24`).

The `server` module owns the loop and the three handlers. `runner.rs` allocates one receive and one
transmit buffer of `HDR_LEN + IPC_PAYLOAD_MAX` each and loops forever (`runner.rs:28`). `respond.rs`
builds a status-only reply or a status-plus-payload reply through `mk_ipc_reply` (`respond.rs:21`,
`respond.rs:27`). The handlers are `health` (reply status 0), `decode` (the format dispatch), and
`surface` (register the ARGB pixels) (`server/handlers/mod.rs:17`).

The `decode` handler is the parser boundary. It allocates `MAX_PIXELS` words, matches the opcode to a
toolkit decoder, and on success calls `register_argb_surface` with the decoded pixel slice and size
(`handlers/decode.rs:26`, `decode.rs:36`). The `surface` handler maps an anonymous private region of
`stride * height` bytes through `mk_mmap`, copies the pixels into it, fills a `SurfaceDescriptor` with the
ARGB8888 format tag, registers it with `mk_surface_register`, and shares it with `mk_surface_share`,
returning the shared handle, stride, and byte length (`handlers/surface.rs:28`). If registration fails it
unmaps the region before returning the error (`surface.rs:49`).

Lifecycle:

1. The desktop services plan calls `spawn_image_codec`, which runs the boot helper with prefix
   `IMAGE-CODEC` and name `image_codec` (`src/userspace/init/spawn_plan/desktop_services.rs:43`,
   `spawn_plan/boot.rs:20`). The helper calls `spawn_image_codec_capsule`, which decodes the baked trust
   anchor and hands the embedded ELF, id cert, manifest, and attestation trailer to `spawn_verified`
   (`src/userspace/capsule_image_codec/spawn.rs:37`, `spawn.rs:53`).
2. `spawn_verified` preflights the id cert and the manifest (publisher signature, capability ceiling,
   target triple, declared endpoints, and the ZK attestation gate), then installs the capsule with the
   caps the manifest declares (`runner/verified.rs:26`, `runner/preflight.rs:29`). It registers
   `image_codec` on port 4412 and the reply inbox on 4413, and emits the `[SPAWN]` line
   (`runner/install/spawn_log.rs:17`).
3. On success the boot helper logs `[IMAGE-CODEC] capsule spawned` and registers the capsule's liveness
   state (`src/userspace/init/capsule_boot/run.rs:29`, `run.rs:30`, `src/userspace/capsule_image_codec/state.rs:21`).
4. The capsule's `_start` initializes its heap and runs the receive-dispatch-reply loop forever
   (`src/main.rs:28`, `server/runner.rs:28`).

The loop is a drain: `drain_ipc` blocks for the first message, then switches to non-blocking receives and
services everything queued until a receive returns nothing, then returns to the blocking wait
(`runner.rs:36`, `runner.rs:40`, `runner.rs:42`). A receive with a zero sender pid is treated as empty and
ends the drain.

## Protocol and IPC

The codec is a server: it registers `image_codec` on service port 4412 with a reply inbox on 4413 and
answers requests. It makes no outbound service calls of its own; its only kernel calls are the IPC receive
and reply (`mk_ipc_recv_from`, `mk_ipc_reply`) and the surface primitives (`mk_mmap`, `mk_munmap`,
`mk_surface_register`, `mk_surface_share`) used to publish the decoded pixels
(`server/runner.rs:19`, `server/respond.rs:17`, `server/handlers/surface.rs:17`).

Request frame, 20-byte header then payload, all little-endian (`protocol/decode.rs`, `protocol/header.rs`):

```
  +0   u32  magic = 0x474D494E ('GMIN')     decode.rs:28  header.rs:17
  +4   u16  version = 1                      decode.rs:35  header.rs:18
  +6   u16  op                               decode.rs:39
  +8   u16  flags                            decode.rs:43
  +10  u16  reserved                         (skipped by the parser)
  +12  u32  request_id                       decode.rs:47
  +16  u32  payload_len                      decode.rs:54
  +20  ...  payload (payload_len bytes)      decode.rs:65
```

The parser requires the whole buffer to be exactly `HDR_LEN + payload_len`; a mismatch is `E_BAD_LEN`
(`decode.rs:62`). The response reuses the header with the same op, flags, and request id, a zeroed
reserved field, and the reply payload length, followed by a 4-byte status word (0 on success) and then any
body (`protocol/encode.rs:19`, `respond.rs:22`, `respond.rs:29`). A decode success writes the 32-byte
descriptor documented above after the status word; every error path writes only the status word
(`respond.rs:21`).

Dispatch in the loop (`runner.rs:51`): a parse error replies with the parser's errno and continues;
`OP_HEALTHCHECK` is served only with an empty body; the four decode ops go to the decode handler; any
other op with an empty body is `E_BAD_OP`, and anything else is `E_INVAL`.

There is no in-tree caller of this service over IPC. The wallpaper capsule decodes PNG in-process with the
same toolkit decoder rather than calling port 4412 (`userland/capsule_wallpaper/src/decode_client/wire.rs:21`,
`wire.rs:29`), so if you are wiring a caller, this page and the `GMIN` frame above are the contract to
target.

## Security analysis

The reason this capsule exists is isolation, and the mask is what makes the isolation worth having. It
parses untrusted image bytes, and the mask bounds what a subverted parser can reach.

Untrusted input, contained blast radius. Every image is attacker-influenced bytes, and the decoders are
real format parsers from `nonos_toolkit::image` running over them (`handlers/decode.rs:18`), which is
exactly the class of code that carries parser bugs. The installed mask `0x1819` has no `Network`, no
`FileSystem`, no `Crypto`, no `Hardware`, and no driver, MMIO, IRQ, DMA, or PIO capability
(`Capsule.mk:11` decomposed against `src/capabilities/types.rs`). A decoder that is subverted by a
malformed image is trapped in a capsule that can receive IPC, allocate memory, query the display, create a
surface, and reply, and nothing else. It cannot read a file, open a socket, or reach a device off the back
of a parser exploit.

Parser-safety posture. The wire parser is bounds-checked end to end: it uses `try_from` on fixed slices,
`checked_add` for the payload end, and requires an exact length match, returning a typed errno on every
failure rather than reading past the buffer (`protocol/decode.rs:25`, `decode.rs:58`, `decode.rs:62`). The
capsule builds with `panic = "abort"` (`Cargo.toml:19`), and the decode path returns errors as status
codes, never a partial surface: a decoder error becomes `fail(...)` with a mapped errno and no handle
(`handlers/decode.rs:34`, `decode.rs:47`). The toolkit decoders check their output against the caller's
buffer before writing (`OutputTooSmall`) and reject zero dimensions (`BadDimensions`)
(`userland/toolkit/src/image/lz4_raw.rs:11`, `image/types.rs:18`).

Bounded output. The fixed `MAX_PIXELS` buffer means a crafted image cannot make the codec allocate an
unbounded pixel buffer (`handlers/decode.rs:23`). The surface allocation multiplies stride and height with
`checked_mul` and returns `E_INVAL` on overflow before it maps anything (`handlers/surface.rs:29`,
`surface.rs:30`). A registration failure unmaps the region rather than leaking it (`surface.rs:49`).

Why `GraphicsSurfaceCreate` and not more. A decode ends by registering the decoded pixels as an ARGB
surface and returning the shared handle (`handlers/surface.rs:48`, `surface.rs:55`), so the caller gets a
surface it can hand to the compositor rather than a raw copy of the pixels back over IPC. The codec does
not hold `GraphicsSurfaceMap` or `GraphicsPresent`: it creates surfaces, it does not map arbitrary ones or
paint the screen.

Stateless. No session is carried between requests, so one caller's image cannot influence another's
decode; the only shared thing produced is the surface, owned by the surface registry once registered.
Isolation from other capsules is the kernel's: the codec is a CPL 3 user binary that only speaks IPC and
its own surfaces, and it is verified and enrolled at spawn like every other capsule
(`runner/preflight.rs:29`).

## How to contribute

The source lives at `userland/capsule_image_codec/`. The wire format is under `src/protocol/`, the loop
and handlers under `src/server/`. The decoders themselves are not in this capsule; they live in the shared
toolkit at `userland/toolkit/src/image/` and are shared with any other capsule that wants to decode in
process.

To add a new format or decoder:

1. Add or extend the decoder in the toolkit at `userland/toolkit/src/image/`, exposing a
   `decode_<fmt>_argb8888(input, out) -> Result<ImageSize, DecodeError>` in the same shape as the existing
   ones (`userland/toolkit/src/image/png/decoder.rs:11`, `image/lz4_raw.rs:3`), and re-export it from
   `image/mod.rs:1`.
2. Add the opcode constant in `src/protocol/ops.rs:17` (the next free value after `OP_DECODE_JPEG =
   0x0005`) and re-export it from `src/protocol/mod.rs:29`.
3. Add the match arm in the decode handler that calls your decoder, and add the opcode to the decode-op
   group in the loop's dispatch (`src/server/handlers/decode.rs:27`, `src/server/runner.rs:53`). If your
   format needs a payload prefix like LZ4, follow the `decode_lz4` shape (`handlers/decode.rs:49`).
4. If your decoder introduces a new `DecodeError` variant, map it to an errno in `map_decode_error`
   (`handlers/decode.rs:56`); otherwise it is covered by the existing mapping.

Building and signing uses the generated per-slug make targets. The slug is `image-codec`
(`Capsule.mk:1`), and the target template is in `nonos-mk/capsule.mk:158`, included through
`userland/capsule_image_codec/Capsule.mk:14`:

```
  make nonos-mk-image-codec              build the capsule ELF          nonos-mk/capsule.mk:182
  make nonos-mk-image-codec-sign         sign the id cert, manifest, and attestation trailer   nonos-mk/capsule.mk:261
  make nonos-mk-image-codec-verify       verify the signed artifacts against the trust anchor  nonos-mk/capsule.mk:263
  make nonos-mk-check-image-codec-keys   assert the per-capsule signing seeds and pubs exist   nonos-mk/capsule.mk:184
```

There is no image-codec-specific `-prod` target; the codec ships as part of the desktop image because its
feature is in the `microkernel-desktop-gui` profile (`Cargo.toml:469`), and its artifacts are pulled into
the desktop image lists in the top-level `Makefile` (`Makefile:576`, `Makefile:726`, `Makefile:881`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every error path returns a typed errno through `respond::status`, and the
release profile is `panic = "abort"`, `Cargo.toml:19`); modular files, one unit per file, with `mod.rs`
used only for re-exports (`src/protocol/mod.rs`, `src/server/handlers/mod.rs`); and the AGPL header at the
top of every source file, matching the header already on every module.

## Debugging

The first thing to confirm is that the capsule ran. Two markers appear on a successful spawn. The install
path prints `[SPAWN] name=image_codec pid=0x... caps=0x1819 entry=0x...`
(`src/kernel_core/process_spawn/capsule_spawn/runner/install/spawn_log.rs:17`), and the boot helper then
prints `[IMAGE-CODEC] capsule spawned` (`src/userspace/init/capsule_boot/run.rs:29`,
`src/sys/boot_log/output.rs:33`). The `caps=0x1819` field is the installed mask; if a build ever changed
the requested or declared caps, that number is where it shows. An absent pair means the capsule never
started, usually a signature, manifest, or capability failure, and the error path prints an `[ERROR]` line
instead (`capsule_boot/run.rs:32`, `boot_log/output.rs:49`).

The failure signatures are on the wire, and they are typed. Because the service is stateless, a codec that
decodes one image and fails the next is behaving correctly on two different inputs, not degrading.

- Frame rejected before decode. A wrong magic returns `E_BAD_MAGIC` (`-74`), a wrong version returns
  `E_BAD_VERSION` (`-93`), and a short header or a payload length that does not match the buffer returns
  `E_BAD_LEN` (`-90`) (`protocol/decode.rs:33`, `decode.rs:52`, `decode.rs:26`, `decode.rs:62`). A caller
  that sees these is looking at a malformed frame, not a bad image.
- Unknown or misused op. An opcode outside 1..5 with an empty body returns `E_BAD_OP` (`-38`); a non-empty
  body on an unknown op returns `E_INVAL` (`-22`) (`runner.rs:54`, `runner.rs:55`).
- Decode failed. A truncated or malformed image, an unsupported feature, or an image that exceeds the
  `MAX_PIXELS` budget returns a typed status and no handle: `E_INVAL` for a bad container magic,
  `E_UNSUPPORTED` for a recognised-but-unsupported feature, and `E_BAD_LEN` for bad dimensions, a
  too-small output, or truncation (`handlers/decode.rs:34`, `decode.rs:47`, `decode.rs:56`). A caller that
  gets a nonzero status and no surface handle is looking at a bad image, not a dead codec.
- Surface allocation failed. If the decode succeeded but the surface could not be mapped, registered, or
  shared, the reply is `E_NOMEM` (`-12`) or the raw negative registration status (`handlers/surface.rs:34`,
  `surface.rs:53`, `surface.rs:57`).
- Payload too large. A request whose framed body would exceed `IPC_PAYLOAD_MAX` (128 KiB) cannot fit the
  receive buffer, so it is bounded at the transport before decode (`runner.rs:29`, `protocol/limits.rs:17`).

## Source map

```
  src/main.rs                              _start -> heap init -> server::run
  src/protocol/header.rs                   GMIN magic, version, 20-byte header, Request struct
  src/protocol/decode.rs                   frame parser (magic, version, exact-length checks)
  src/protocol/encode.rs                   response header and status word writers
  src/protocol/ops.rs                      the five opcodes (healthcheck + four decodes)
  src/protocol/errno.rs                    the typed error codes
  src/protocol/limits.rs                   IPC_PAYLOAD_MAX, DECODE_RESP_LEN, DECODE_LZ4_PREFIX_LEN
  src/server/runner.rs                     the receive/dispatch/reply drain loop
  src/server/respond.rs                    status-only and status+payload replies over mk_ipc_reply
  src/server/handlers/decode.rs            format dispatch, MAX_PIXELS, the 32-byte reply, error mapping
  src/server/handlers/surface.rs           mmap + register + share the decoded ARGB surface
  src/server/handlers/health.rs            healthcheck reply
  Capsule.mk                               slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_image_codec/       the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/desktop_services.rs   the desktop-fleet spawn entry
  userland/toolkit/src/image/              the shared png, bmp, jpeg, and lz4_raw decoders
  nonos-mk/capsule.mk                      the generated nonos-mk-image-codec[-sign|-verify] targets
```

Every reference above is verified against those trees.

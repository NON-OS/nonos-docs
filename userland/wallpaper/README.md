# capsule_wallpaper

`capsule_wallpaper` renders the desktop background. It owns one full-screen surface at the bottom of the
compositor's Z order, reads the selected wallpaper index from the [policy store](../policy/README.md), streams that
image out of the [wallpaper catalog](../wallpaper-catalog/README.md) in bounded chunks, decodes and paints it into
its backing surface, and asks the compositor to composite the result. It also answers a small control
protocol for setting a flat background color, changing the fit policy, and fading the background alpha.
The source is `userland/capsule_wallpaper/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Behavior reference](#behavior-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The wallpaper capsule is a background painter. It is `no_std`/`no_main`; `_start` initializes the heap,
runs setup, and then enters the server loop that never returns
(`userland/capsule_wallpaper/src/main.rs:37`). Setup discovers the compositor, allocates a backing
surface the size of the display, registers that surface at the bottom Z layer, and primes it with an
image; the loop then drains its own control IPC, polls the policy store for a wallpaper change, and paces
any active fade against the display's vsync clock (`src/server/runner/entry.rs:30`).

Everything the capsule paints goes into a private surface it owns. It never presents to the framebuffer
itself: it writes ARGB pixels into the mapped backing buffer and asks the compositor to commit the
damaged region, and the compositor composites the wallpaper under every window. That single fact is the
whole shape of the security analysis below.

## Identity

Everything the kernel and the service registry need to name and reach the wallpaper capsule comes from
its `Capsule.mk` and the kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `wallpaper` | `Capsule.mk:1` |
| Service handle | `wallpaper` | `Capsule.mk:2`, `src/userspace/capsule_wallpaper/spawn.rs:31` |
| Namespace | `systems.nonos.wallpaper` | `Capsule.mk:7` |
| Service endpoint | `service:4340:wallpaper` | `Capsule.mk:8`, `spawn.rs:31`, `spawn.rs:32` |
| Reply endpoint | `reply:4341:endpoint.wallpaper.reply` | `Capsule.mk:9`, `spawn.rs:33`, `spawn.rs:34` |
| Capability mask | `0x1819` | `Capsule.mk:12` |
| Binary name | `wallpaper` | `Capsule.mk:5` |
| Kernel mirror | `src/userspace/capsule_wallpaper` | `Capsule.mk:13` |

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

The kernel spawn path requests exactly those five capabilities and no others
(`src/userspace/capsule_wallpaper/spawn.rs:50`). There is no `Network` bit (4), no `FileSystem` bit (64),
and no hardware, driver, or DMA capability, and in particular no `GraphicsPresent` bit
(`0x4000`, `types.rs:70`). It can create a surface, ask the display for its geometry, and speak IPC; it
cannot present to the screen, which is what keeps the compositor in the middle.

## Behavior reference

### Selecting the wallpaper

The wallpaper capsule does not decide which image to show; the [policy store](../policy/README.md) does. The store
keeps a single `wallpaper: u8` field (policy `Field::Wallpaper`, discriminant `0x0117`,
`userland/policy_proto/src/field.rs:42`), and the wallpaper client reads it with an `OP_GET` of kind
`KIND_U8` (`src/policy_client/get_wallpaper.rs:22`). The returned byte is a flat catalog index: it is
passed straight to the catalog fetch. When a user changes the wallpaper in the settings panel, the panel
writes that policy field and the wallpaper capsule re-reads it and re-fetches; the catalog default index
is documented in the [wallpaper catalog](../wallpaper-catalog/README.md) page.

Selection happens in two places:

- At setup, once the surface exists, `run` reads the policy field and applies it immediately, before the
  live session starts, so the first catalog decode and full-screen recomposite do not stall the single
  core a few seconds into boot (`src/setup/prime/run.rs:62`). If that succeeds the index is recorded in
  `applied_wallpaper` so the subscriber will not repeat the work.
- During the session, the subscriber polls the policy store every 300 pacer ticks
  (`src/subscriber/tick.rs:23`). If the store's `wallpaper` value differs from `applied_wallpaper`, it
  re-fetches, re-decodes, re-paints, and records the new index (`src/subscriber/tick.rs:44`). A poll that
  matches the applied index is a no-op, so a changed wallpaper redraws and an unchanged one does not.

### Fetching image bytes from the catalog

`apply` turns an index into pixels (`src/subscriber/apply.rs:22`). It looks up the catalog port, calls
`fetch_image` for the index, decodes the JPEG, paints it, and commits damage. `fetch_image`
(`src/catalog_client/fetch_image.rs:26`) is the streaming client: it calls `OP_GET_SIZE` once, rejects a
size of zero or over 2,000,000 bytes, then loops `OP_GET_CHUNK` advancing the offset by each reply's
`payload_len` until it has the whole image, bounding the loop by a chunk count and rejecting a reassembled
length that does not match the declared size (`fetch_image.rs:28`, `fetch_image.rs:37`,
`fetch_image.rs:49`). The size and chunk calls each have a 500 ms reply timeout and validate that the
reply echoes the requested op, index, and offset (`src/catalog_client/fetch_size.rs:39`,
`src/catalog_client/fetch_chunk.rs:38`). The catalog protocol is described in full on the
[wallpaper catalog](../wallpaper-catalog/README.md) page.

### Decoding and painting

Two decode paths exist, and both run in-process:

- `decode_jpeg` (`src/paint/decode_jpeg.rs:33`) is used for catalog images and the embedded boot image.
  It checks the `FF D8` JPEG marker, decodes into a `1920x1080` scratch buffer through
  `nonos_toolkit::image::jpeg::decode_jpeg_argb8888`, and rejects a zero, oversized, or over-`MAX_PIXELS`
  result (`decode_jpeg.rs:42`). `paint_image` then nearest-neighbor stretches the decoded pixels across
  the whole backing surface (`src/paint/paint_image.rs:22`, `src/paint/blit_argb.rs:31`).
- The `decode_client` (`src/decode_client/`) handles an image carried inline on a `SET_WALLPAPER`
  request. It parses a four-field decode header (kind, width, height, payload length,
  `src/decode_client/header.rs:36`), decodes PNG, BMP, raw LZ4, or JPEG through the toolkit
  (`src/decode_client/wire.rs:27`), and stretches the result into the backing surface
  (`src/decode_client/seq.rs:39`).

A flat color path bypasses decoding: `set_argb` records the color and `fill_argb` writes it across every
pixel of the backing surface (`src/state/context.rs:43`, `src/paint/fill.rs:20`). The default at setup is
`0xFF0080FF` (`src/setup/prime/run.rs:25`), painted before the embedded `special-variant-6-1080p.jpg` is
decoded over it (`src/setup/prime/run.rs:26`, `src/setup/prime/run.rs:51`).

Note an honest gap: the `Policy` fit style is stored but not consulted by either paint path. `Policy`
has `Fill`, `Fit`, `Stretch`, `Center`, and `Tile` variants (`src/state/policy.rs:19`), and
`SET_POLICY` records the selected one (`src/server/handlers/set_policy.rs:34`), but both
`blit_argb` and the decode-client painter always nearest-neighbor stretch the image to the full backing
dimensions regardless of the stored policy (`src/paint/blit_argb.rs:31`, `src/decode_client/seq.rs:51`).
The fit style is captured and reported through `GET_WALLPAPER` but does not yet change how pixels land.

### Redraw on change

Every path that changes what should be on screen paints the backing buffer and then commits damage to the
compositor over the whole surface: the subscriber after a catalog swap (`src/subscriber/apply.rs:37`),
the `SET_WALLPAPER` handler after a color or inline image (`src/server/handlers/set_wallpaper.rs:61`), and
the fade pacer on each alpha step (`src/server/tick.rs:41`). A `FADE` request starts a linear alpha ramp
sampled against the vsync clock; the runner's pacer tick reads the interpolated alpha each frame, repaints
the flat color at the new alpha, and commits damage until the ramp completes
(`src/state/fade.rs:53`, `src/server/tick.rs:25`). A zero-duration fade snaps to the target immediately
rather than animating (`src/state/fade.rs:36`, `src/server/handlers/fade.rs:46`).

## Architecture and lifecycle

The capsule is `no_std`/`no_main` with nine top-level modules: `catalog_client`, `compositor_client`,
`decode_client`, `paint`, `policy_client`, `protocol`, `server`, `setup`, and `state`
(`src/main.rs:22`). The runtime `Context` (`src/state/context.rs:19`) is the whole live state: the
compositor port, the backing surface geometry and base address, the current color and alpha, the fit
policy, the fade timeline, a monotonically issued request id, the policy and catalog ports (each an
`Option` resolved lazily), the last applied wallpaper index, and the subscriber tick counter.

Lifecycle:

1. `_start` initializes the heap, exiting with code 1 on failure, then calls `wait_for_setup`, which
   retries `setup::run` with a yield backoff until it succeeds (`src/main.rs:38`,
   `src/wait_for_setup.rs:19`).
2. Setup discovers the compositor by name, retrying up to 256 times and giving up with
   `"compositor service not announced"` if the lookup never resolves (`src/setup/discover.rs:32`). It then
   healthchecks the compositor, queries the display geometry, and `mmap`s a backing buffer sized
   `stride * height` (`src/setup/prime/run.rs:30`, `src/setup/prime/backing.rs:32`).
3. It fills the buffer with the default color, decodes and paints the embedded boot JPEG, then registers
   the surface: `mk_surface_register`, `mk_surface_share`, and a `scene_submit` to the compositor at Z 0
   (`src/setup/prime/register.rs:41`). The policy and catalog ports are looked up here, tolerating
   absence.
4. Setup applies the policy-selected wallpaper synchronously if the policy port resolved, marking it
   applied (`src/setup/prime/run.rs:62`).
5. The server loop drains control IPC non-blocking, runs the subscriber poll, and then either paces an
   active fade or waits for vsync when idle (`src/server/runner/entry.rs:30`).

## Protocol and IPC

The wallpaper capsule both serves a control protocol and calls three peer services.

### The control protocol it serves

Requests arrive on the service inbox and are parsed against a 20-byte header carrying magic `NWLP`
(`0x4E574C50`) and version 1 (`src/protocol/header.rs:17`, `src/protocol/decode.rs:21`). The parser
rejects a short frame with `E_BAD_LEN`, a wrong magic with `E_BAD_MAGIC`, a wrong version with
`E_BAD_VERSION`, and a body length that does not match the header with `E_BAD_LEN`
(`src/protocol/decode.rs:22`). Five operations dispatch (`src/protocol/ops.rs:17`,
`src/server/runner/dispatch.rs:24`):

```
  OP_HEALTHCHECK    = 0x0001   reply status 0
  OP_SET_WALLPAPER  = 0x0002   set flat color or paint an inline image
  OP_GET_WALLPAPER  = 0x0003   report color, policy, dimensions, alpha
  OP_SET_POLICY     = 0x0004   select the fit style
  OP_FADE           = 0x0005   start an alpha ramp
```

| Op | Body | Effect | Handler |
|---|---|---|---|
| `OP_HEALTHCHECK` | empty | reply status 0 | `handlers/health.rs:20` |
| `OP_SET_WALLPAPER` | 8-byte color `argb,_pad`, or a decode-client image | set color or paint image, commit damage | `handlers/set_wallpaper.rs:43` |
| `OP_GET_WALLPAPER` | empty | reply `argb, policy, width, height, alpha, _pad` (24 bytes) | `handlers/get_wallpaper.rs:21` |
| `OP_SET_POLICY` | 8-byte `policy,_pad` | record the fit style, or `E_INVAL` for an unknown one | `handlers/set_policy.rs:21` |
| `OP_FADE` | 8-byte `target_alpha, duration_ms` | start a ramp, or `E_INVAL` if alpha exceeds 255 | `handlers/fade.rs:25` |

`SET_WALLPAPER` is gated: only `desktop_shell` and `policy`, resolved from their service names to pids
through the registry, may set it; any other caller gets `E_ACCES` (`handlers/set_wallpaper.rs:26`,
`handlers/set_wallpaper.rs:44`). An unknown op with an empty body replies `E_BAD_OP`; an unknown op with a
body replies `E_INVAL` (`src/server/runner/dispatch.rs:33`). Replies are framed by `respond::status` and
`respond::payload`, which reuse the request's op, flags, and request id and write a little-endian status
word (`src/server/respond.rs:21`, `src/protocol/encode.rs:19`). The full errno set is `E_ACCES` (-13),
`E_INVAL` (-22), `E_BAD_OP` (-38), `E_BAD_MAGIC` (-71), `E_BAD_LEN` (-90), and `E_BAD_VERSION` (-93)
(`src/protocol/errno.rs:17`).

### The catalog it calls

Service `wallpaper_catalog`, resolved by name (`src/catalog_client/lookup.rs:23`), spoken with a 16-byte
compact header (`src/catalog_client/proto.rs:28`), 500 ms reply timeout:

```
  OP_GET_SIZE   = 0x0002   fetch_size.rs:25    image byte length as a u32
  OP_GET_CHUNK  = 0x0003   fetch_chunk.rs:24   up to 4096 bytes at an offset
```

The size and chunk semantics and the errno set on that side are documented on the
[wallpaper catalog](../wallpaper-catalog/README.md) page.

### The policy store it reads

Service `policy` (the shared `nonos_policy_proto`), resolved by name
(`src/policy_client/lookup.rs:22`), 200 ms reply timeout: a single `OP_GET` of `Field::Wallpaper` with
kind `KIND_U8`, validating that the reply echoes the op, kind, and field (`src/policy_client/get_wallpaper.rs:22`).

### The compositor it drives

Service `compositor`, spoken with a 20-byte `NCMP` header (`0x4E434D50`, version 1,
`src/compositor_client/wire.rs:10`). The calls it makes:

```
  OP 0x0001  healthcheck    compositor_client/health.rs:19        250 ms boot timeout
  OP 0x0008  display_info   compositor_client/display_info.rs:21  query width/height/stride/format
  OP 0x0002  scene_submit   compositor_client/scene_submit.rs:21  register the surface handle at a Z
  OP 0x0003  damage_commit  compositor_client/damage_commit.rs:20 commit a damaged rectangle
```

`healthcheck`, `display_info`, and `scene_submit` run on the boot timeout during setup; `damage_commit`
runs on the tighter 16 ms live timeout (`src/compositor_client/wire.rs:13`). Each reply is validated to
echo the magic, version, op, request id, and payload length (`src/compositor_client/wire/reply.rs:40`).

## Security analysis

Its mask `0x1819` grants CoreExec, IPC, Memory, GraphicsDisplayQuery, and GraphicsSurfaceCreate
(`Capsule.mk:12`, `src/userspace/capsule_wallpaper/spawn.rs:50`). There is no Network bit, no FileSystem
bit, no hardware, driver, MMIO, or DMA capability, and no `GraphicsPresent`. It has
`GraphicsSurfaceCreate` to register the backing surface it owns and `GraphicsDisplayQuery` to size it
(`src/setup/prime/register.rs:41`, `src/setup/prime/backing.rs:33`).

- **A background painter and nothing more.** The mask lets the capsule draw into one surface it owns and
  ask the compositor to composite it. Because it lacks `GraphicsPresent`, it cannot put pixels on the
  screen directly, cannot map another capsule's surface, and cannot touch a device, so a compromise is
  bounded to painting garbage into the background layer, which the compositor still keeps under every
  window.
- **Write access to the background is gated.** Even the control protocol is not open: `SET_WALLPAPER`,
  the one op that changes what is painted from outside, is restricted to `desktop_shell` and `policy` by
  resolving their service names to pids, and every other caller gets `E_ACCES`
  (`src/server/handlers/set_wallpaper.rs:26`). The read-only `GET_WALLPAPER`, `SET_POLICY`, `FADE`, and
  `HEALTHCHECK` are open to any caller with IPC rights, but none of them can substitute an arbitrary
  image; `SET_POLICY` only records a fit style, and `FADE` only ramps alpha.
- **Image bytes are decoded in-process, which is the honest boundary.** Both catalog images and inline
  `SET_WALLPAPER` images are decoded by `nonos_toolkit` inside this capsule (`src/paint/decode_jpeg.rs`,
  `src/decode_client/wire.rs`). There is no separate image-codec service on the wallpaper's path: the
  `decode_client` name notwithstanding, it makes no IPC call and holds no codec service port. That means a
  real image parser runs over untrusted-length JPEG/PNG/BMP/LZ4 bytes inside the capsule that also owns
  the background surface. The decoders bound their output buffers and reject malformed or oversized inputs
  (`src/paint/decode_jpeg.rs:42`, `src/catalog_client/fetch_image.rs:28`), but the parser exposure lives
  in this capsule rather than in an isolated one, and that is the trust boundary to keep in mind.

Isolation from other capsules is the kernel's: the wallpaper is a CPL 3 user binary that speaks only IPC
and its own surface, and it is verified and enrolled at spawn like every other capsule
(`src/userspace/capsule_wallpaper/spawn.rs:40`).

## How to contribute

The source lives at `userland/capsule_wallpaper/`. To change how images are scaled, edit the paint path:
`src/paint/blit_argb.rs` for catalog and embedded images and `src/decode_client/seq.rs` for inline
images; both currently stretch to the full backing size and neither consults `src/state/policy.rs`, so
wiring the fit policy in is the natural first task. To change how images are fetched, edit the streaming
client under `src/catalog_client/`. To change selection, edit `src/policy_client/get_wallpaper.rs` and the
subscriber under `src/subscriber/`.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_wallpaper/Capsule.mk:15`):

```
  make nonos-mk-wallpaper               build the capsule ELF
  make nonos-mk-wallpaper-sign          produce the id cert, manifest, and attestation trailer
  make nonos-mk-wallpaper-verify        verify the signed artifacts against the trust anchor
  make nonos-mk-check-wallpaper-keys    check the per-capsule signing keys exist
```

The `-sign`, `-verify`, and `-keys` targets are generated by the `define` block in
`nonos-mk/capsule.mk:158`; `make nonos-mk-wallpaper` is also listed explicitly in the top-level `.PHONY`
(`Makefile:31`). There is no wallpaper-specific `-prod` target; the wallpaper is built and signed as part
of the desktop GUI image alongside the other fleet capsules (`Makefile:648`, `Makefile:681`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns errors as a status word, never a panic; the release and
dev profiles are both `panic = "abort"`, `Cargo.toml:31`, `Cargo.toml:38`); modular files, one unit per
file, with `mod.rs` used only for re-exports; and the AGPL header at the top of every source file,
matching the header on every existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints `[WALLPAPER]
capsule spawned` from the desktop-fleet spawn (`src/userspace/init/spawn_plan/desktop_fleet.rs:109`,
`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line means the
capsule never started, usually a signature, manifest, or capability failure, and an `[ERROR]` line is
printed instead (`src/userspace/init/capsule_boot/run.rs:32`). The kernel spawn requests exactly
`caps=0x1819` (`src/userspace/capsule_wallpaper/spawn.rs:50`).

Failure modes and where to look:

- Blank or default-colored background, no image. The default fill color `0xFF0080FF` is painted first at
  setup, so a solid blue-ish background means the embedded and policy-selected images never landed
  (`src/setup/prime/run.rs:25`). The usual cause is the catalog or policy port not resolving: both are
  looked up tolerantly and left `None` on failure, so the subscriber degrades to leaving the color rather
  than failing setup (`src/setup/prime/run.rs:45`, `src/subscriber/tick.rs:30`). Confirm `wallpaper_catalog`
  and `policy` are registered.
- Wrong wallpaper. The index comes only from the policy `wallpaper` field
  (`src/policy_client/get_wallpaper.rs:22`); a wrong image means a wrong stored index, not a wallpaper
  bug. Cross-check the stored value against the catalog's slug for that index, described on the
  [wallpaper catalog](../wallpaper-catalog/README.md) page.
- The wallpaper never registers, so the desktop comes up without a background layer. Setup blocks on the
  compositor, retrying the lookup and giving up with `"compositor service not announced"` only after 256
  tries (`src/setup/discover.rs:32`); a surface register or share rejection surfaces as
  `"surface register rejected"` or `"surface share rejected"` (`src/setup/prime/register.rs:43`,
  `src/setup/prime/register.rs:47`).
- A wallpaper that snaps instead of dissolving. A `FADE` with a zero duration is defined to snap to the
  target alpha rather than animate (`src/state/fade.rs:36`, `src/server/handlers/fade.rs:46`), so that is
  a zero-duration request, not a fault.
- A `SET_WALLPAPER` that is ignored with `E_ACCES`. The caller is not `desktop_shell` or `policy`; the
  write gate is doing its job (`src/server/handlers/set_wallpaper.rs:44`).

## Source map

```
  userland/capsule_wallpaper/src/main.rs                    heap init, wait_for_setup, server::run
  userland/capsule_wallpaper/src/setup/discover.rs          compositor wait, "compositor service not announced"
  userland/capsule_wallpaper/src/setup/prime/run.rs         default color, embedded image, register, apply policy
  userland/capsule_wallpaper/src/setup/prime/backing.rs     display_info + mmap the backing surface
  userland/capsule_wallpaper/src/setup/prime/register.rs    surface register/share/scene_submit at Z 0
  userland/capsule_wallpaper/src/server/runner/             drain IPC, subscriber tick, fade pacer, vsync wait
  userland/capsule_wallpaper/src/server/handlers/           health, set_wallpaper, get_wallpaper, set_policy, fade
  userland/capsule_wallpaper/src/server/respond.rs          status and payload reply framing
  userland/capsule_wallpaper/src/protocol/                  NWLP header, ops, errnos, limits, parse/encode
  userland/capsule_wallpaper/src/state/context.rs           color, alpha, policy, fade, ports, applied index
  userland/capsule_wallpaper/src/state/fade.rs              the linear alpha ramp
  userland/capsule_wallpaper/src/state/policy.rs            the fit-style enum (stored, not yet applied)
  userland/capsule_wallpaper/src/subscriber/                policy poll, apply: fetch + decode + paint + commit
  userland/capsule_wallpaper/src/catalog_client/            OP_GET_SIZE / OP_GET_CHUNK image streaming
  userland/capsule_wallpaper/src/policy_client/             OP_GET Field::Wallpaper
  userland/capsule_wallpaper/src/compositor_client/         health, display_info, scene_submit, damage_commit
  userland/capsule_wallpaper/src/paint/                     fill_argb, decode_jpeg, blit_argb, paint_image
  userland/capsule_wallpaper/src/decode_client/             inline image decode (png/bmp/lz4/jpeg, in-process)
  userland/capsule_wallpaper/Capsule.mk                     slug, handle, ports 4340/4341, caps 0x1819
  src/userspace/capsule_wallpaper/spawn.rs                  the kernel-side verified spawn, requested caps
  src/userspace/init/spawn_plan/desktop_fleet.rs            the desktop-fleet spawn entry
  nonos-mk/capsule.mk                                       the generated nonos-mk-wallpaper[-sign|-verify] targets
```

Every reference above is verified against those trees.
</content>
</invoke>

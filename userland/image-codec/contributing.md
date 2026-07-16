# Contributing

This page covers where to work in the codec, how to add a format or decoder, and how to build and sign the
capsule. The source lives at `userland/capsule_image_codec/`. The wire format is under `src/protocol/`
([protocol](protocol.md)), the loop and reply machinery under `src/server/` ([server](server.md)), and the
handlers under `src/server/handlers/` ([decode](decode.md)). The decoders themselves are not in this
capsule: they live in the shared toolkit at `userland/toolkit/src/image/` and are shared with any other
capsule that decodes in-process.

## Where to work

| You want to change | Work in |
|--------------------|---------|
| The frame, an opcode, an error code, or a limit | `src/protocol/` |
| The loop, dispatch, or the reply builders | `src/server/runner.rs`, `src/server/respond.rs` |
| The format dispatch or the surface path | `src/server/handlers/decode.rs`, `handlers/surface.rs` |
| A decoder itself, or a new format's parser | `userland/toolkit/src/image/` |
| The service identity, ports, or mask | `Capsule.mk` |

## Adding a format or decoder

Adding a fifth decode op takes four steps that mirror how the existing four are wired.

1. Add or extend the decoder in the toolkit at `userland/toolkit/src/image/`, exposing a
   `decode_<fmt>_argb8888(input, out) -> Result<ImageSize, DecodeError>` in the same shape as the existing
   ones (`userland/toolkit/src/image/png/decoder.rs:11`, `bmp.rs:13`, `lz4_raw.rs:3`), and re-export the
   submodule from `userland/toolkit/src/image/mod.rs:1`. Have the decoder check its output against the
   caller's buffer length and reject zero or oversized dimensions, the way the existing decoders do
   (`userland/toolkit/src/image/lz4_raw.rs:9`, `lz4_raw.rs:11`, `userland/toolkit/src/image/types.rs:1`);
   the [safety](safety.md) page explains why that bound matters.

2. Add the opcode constant in `src/protocol/ops.rs:17`. Use the next free value after `OP_DECODE_JPEG =
   0x0005` (`ops.rs:21`) and re-export it from `src/protocol/mod.rs:29`.

3. Add the match arm in the decode handler that calls your decoder (`src/server/handlers/decode.rs:27`),
   and add the new opcode to the decode-op group in the loop's dispatch so the loop routes it to the decode
   handler (`src/server/runner.rs:53`). If your format needs a payload prefix like LZ4, follow the
   `decode_lz4` shape, which parses the prefix before calling the decoder
   (`src/server/handlers/decode.rs:49`).

4. If your decoder introduces a new `DecodeError` variant, map it to an errno in `map_decode_error`
   (`src/server/handlers/decode.rs:56`); otherwise the existing mapping covers it. If you need a new errno,
   define it in `src/protocol/errno.rs` and re-export it from `src/protocol/mod.rs:26`.

Nothing in the identity needs to change to add an op: the mask already grants what a decode needs
(`Capsule.mk:11`), and the reply descriptor is format-independent (`src/protocol/limits.rs:20`).

## Building and signing

The build and sign targets are generated per slug by `nonos-mk`. The slug is `image-codec`
(`Capsule.mk:1`), and the target template is defined in `nonos-mk/capsule.mk` and pulled in through the
`include` on `Capsule.mk:14`. The generated targets are declared at `nonos-mk/capsule.mk:158`:

```
  make nonos-mk-image-codec              build the capsule ELF                          nonos-mk/capsule.mk:7
  make nonos-mk-image-codec-sign         sign the id cert, manifest, and trailer        nonos-mk/capsule.mk:261
  make nonos-mk-image-codec-verify       verify the signed artifacts vs the trust anchor  nonos-mk/capsule.mk:263
  make nonos-mk-check-image-codec-keys   assert the per-capsule signing seeds and pubs exist  nonos-mk/capsule.mk:10
```

The manifest is signed from `Capsule.mk` with the required caps declared and no optional caps, so the
installed mask resolves to exactly `0x1819` (`Capsule.mk:11`; see [safety](safety.md) for how the install
computes it). There is no image-codec-specific production target: the codec ships as part of the desktop
image because its feature `nonos-capsule-image-codec` is in the desktop profiles (`Cargo.toml:84`,
`Cargo.toml:469`).

## Code standards

- `cargo fmt` and a clean `cargo clippy`.
- No panics, `unwrap`, or `expect` in capsule code. Every error path returns a typed errno through
  `respond::status` (`src/server/respond.rs:21`), and both release profiles are `panic = "abort"`
  (`Cargo.toml:19`, `Cargo.toml:26`).
- Modular files, one unit per file, with `mod.rs` used only for re-exports (`src/protocol/mod.rs:17`,
  `src/server/handlers/mod.rs:17`).
- The AGPL header at the top of every source file, matching the header already on every module.

## Source map

This page is drawn from `userland/capsule_image_codec/` (the `src/` tree and `Capsule.mk`, `Cargo.toml`),
the shared decoders at `userland/toolkit/src/image/`, and the generated make targets in
`nonos-mk/capsule.mk`, together with the desktop feature lists in the top-level `Cargo.toml`. Every
reference above is verified against those trees.

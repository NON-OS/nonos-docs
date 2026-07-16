# capsule_crypto (full reference)

`capsule_crypto` is the userland cryptographic compute pool: a stateless service that takes a hash,
MAC, KDF, signature-verify, AEAD, or ECDH request, computes it, replies, and wipes the request buffer.
It holds no keys and no session state. It also serves a second purpose worth stating up front: it is
where the userland's cryptographic-crate dependencies are concentrated, so that other capsules reach
Ed25519, AES-GCM, and X25519 through IPC rather than each linking a crypto library of its own. This is
the exhaustive reference; the source is `userland/capsule_crypto/`.

The kernel spawns it under service handle `crypto_pool` on service port 4102 with a reply port on 4103,
and its capability mask is `0x39` (`userland/capsule_crypto/Capsule.mk:16`). The important nuance, and
the whole basis of the security section below, is that the mask on the capsule is not the caller-facing
gate. `CAP_CRYPTO` is checked on the kernel side of every request, against the calling pid, before the
capsule ever sees the bytes; the capsule's own `0x39` only lets it run, speak IPC, and drive the
primitives (`Capsule.mk:1`).

## Contents

- [Overview and role](#overview-and-role)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview and role

The design goal is that no other capsule needs a crypto capability or a crypto crate of its own. A caller
that wants a signature checked, a record sealed, or a key derived sends a framed request to `crypto_pool`
and reads back a status and a result. The capsule is a pure function of each request: it decodes the
frame, dispatches to a handler, computes, encodes the reply, sends it, and volatile-zeroes every received
byte before waiting for the next message (`src/server/runner.rs:29`). Nothing survives a request, so a
request never depends on a prior one and there is no state to leak between callers.

Two layers sit in front of the capsule and are easy to conflate, so it is worth naming them. A userland
caller does not talk to the capsule directly; it calls a thin `nonos_libc` shim such as
`crypto_ed25519_verify` (`userland/libc/src/crypto/ed25519_verify.rs:26`), which issues a syscall. The
kernel's crypto syscall dispatch routes that into the crypto-capsule client under `src/security/crypto_capsule/`,
and that client is where `CAP_CRYPTO` is enforced against the caller pid (`capability.rs:22`) before the
request is marshalled and sent over IPC to this capsule. So "the crypto capsule" (this page) and "the
kernel crypto stack" (documented under [crypto subsystem](../../subsystems/crypto/README.md)) are two
distinct bodies of code, and the client that gates and transports requests is a third. This page
documents the capsule.

## Identity

Everything the kernel and the service registry need to name and reach the capsule comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `crypto` | `Capsule.mk:6` |
| Service handle | `crypto_pool` | `Capsule.mk:13`, `src/security/crypto_capsule/spawn.rs:31` |
| Namespace | `systems.nonos.crypto` | `Capsule.mk:12` |
| Service endpoint | `service:4102:crypto_pool` | `Capsule.mk:13`, `spawn.rs:32` |
| Reply endpoint | `reply:4103:endpoint.4294967300` | `Capsule.mk:14`, `spawn.rs:33`, `spawn.rs:47` |
| Reply inbox (kernel client) | `endpoint.4294967300` = `0x1_0000_0004` | `client/transport.rs:25`, `client/transport.rs:28` |
| Capability mask | `0x39` | `Capsule.mk:16` |
| Binary name | `crypto` | `Capsule.mk:10` |
| Kernel mirror | `src/security/crypto_capsule` | `Capsule.mk:17` |

The capsule's own reply endpoint (`src/protocol/types.rs:62`) is `0x1_0000_0004`, which is 4294967300 in
decimal, deliberately distinct from ramfs (4294967297), keyring (4294967298), and entropy (4294967299) so
concurrent in-flight requests to different pools cannot cross-route (`types.rs:60`).

The mask `0x39` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x0001  CoreExec   bit()  1     types.rs:56
  0x0008  IPC        bit()  8     types.rs:59
  0x0010  Memory     bit() 16     types.rs:60
  0x0020  Crypto     bit() 32     types.rs:61
  ------
  0x0039  = 1 + 8 + 16 + 32
```

The kernel spawn path requests exactly `IPC | Memory | Crypto` (`src/security/crypto_capsule/spawn.rs:54`);
`Capsule.mk` adds `CoreExec` implicitly as every capsule's execute bit, so the manifest's required-caps is
`0x39`. There is no `Network` bit (4), no `FileSystem` bit (64), no `Debug` (256), and no graphics,
driver, MMIO, IRQ, DMA, or PIO capability. `Crypto` (32) is held not as a caller gate but because the
capsule drives crypto primitives; the comment in `Capsule.mk:1` states plainly that `CAP_CRYPTO` is the
caller-facing gate and the capsule itself does not hold it in that sense.

## Operation reference

The op table is split across two files, which is a known trap. The first ten opcodes and their sizing
constants are in `src/protocol/types.rs:20`; the remaining seven were folded in later and live in
`src/protocol/primitives.rs:17`. The complete set is seventeen operations, all routed by one match in
`src/server/dispatch.rs:29`. An unknown op is `EINVAL` (`dispatch.rs:47`).

Every frame, request and reply, is the same 20-byte header (`src/protocol/types.rs:64`,
`src/protocol/decode.rs:27`, `src/protocol/encode.rs:21`), the same shape as
[capsule_entropy](../entropy/README.md): `u32 magic ("NOCX" = 0x4E4F4358), u16 version (1), u16 op, u16 flags, u16
reserved, u32 request_id, u32 payload_len`. A reply payload is a leading `i32 status` followed by the
result body (`encode.rs:22`); status `0` is success, and the error codes are `EIO -5`, `EINVAL -22`,
`EBADMSG -74`, `EMSGSIZE -90` (`src/protocol/errno.rs:17`). The decoder rejects a short buffer, a bad
magic, a wrong version, or a `payload_len` over `MAX_PAYLOAD_BYTES` before any handler runs
(`decode.rs:28`).

The complete op table, each opcode and its handler cited:

| Op | Name | Handler | Request payload | Reply body |
|---|---|---|---|---|
| 1 | BLAKE3_HASH | `handlers/blake3_hash.rs:23` | input bytes, `<= 64 KiB` | 32-byte digest |
| 2 | SHA3_256_HASH | `handlers/sha3_256_hash.rs:23` | input bytes, `<= 64 KiB` | 32-byte digest |
| 3 | HEALTHCHECK | `handlers/healthcheck.rs:21` | none | empty (status 0) |
| 4 | SHA256_HASH | `handlers/sha256_hash.rs:23` | input bytes, `<= 64 KiB` | 32-byte digest |
| 5 | SHA512_HASH | `handlers/sha512_hash.rs:23` | input bytes, `<= 64 KiB` | 64-byte digest |
| 6 | ED25519_VERIFY | `handlers/ed25519_verify.rs:38` | `pubkey[32] \|\| sig[64] \|\| message`, message `<= 1 MiB` | empty (status only) |
| 10 | CHACHA20_POLY1305_SEAL | `handlers/chacha20_poly1305_seal.rs:25` | AEAD seal frame (below) | ciphertext+tag |
| 11 | CHACHA20_POLY1305_OPEN | `handlers/chacha20_poly1305_open.rs:25` | AEAD open frame (below) | plaintext |
| 12 | AES256_GCM_SEAL | `handlers/aes256_gcm_seal.rs:25` | AEAD seal frame (below) | ciphertext+tag |
| 13 | AES256_GCM_OPEN | `handlers/aes256_gcm_open.rs:25` | AEAD open frame (below) | plaintext |
| 14 | X25519_PUBLIC | `handlers/x25519_public.rs:22` | `private[32]` | `public[32]` |
| 15 | X25519_SHARED | `handlers/x25519_shared.rs:23` | `private[32] \|\| peer_public[32]` | `shared[32]` |
| 16 | HMAC_SHA256 | `handlers/hmac_sha256.rs:22` | `u32 key_len \|\| key \|\| message`, key `<= 256 B` | 32-byte MAC |
| 17 | HKDF_SHA256 | `handlers/hkdf_sha256.rs:24` | header + salt/ikm/info (below) | derived key |
| 18 | P256_ECDSA_VERIFY | `handlers/p256_ecdsa_verify.rs:22` | `sec1_pubkey[65] \|\| sig[64] \|\| prehash[32]` | empty (status only) |
| 19 | P384_ECDSA_VERIFY | `handlers/p384_ecdsa_verify.rs:22` | `sec1_pubkey[97] \|\| sig[96] \|\| prehash[48]` | empty (status only) |
| 20 | SHA384_HASH | `handlers/sha384_hash.rs:23` | input bytes, `<= 64 KiB` | 48-byte digest |

Opcodes 1 through 6 and 10 through 13 are declared in `types.rs:20`; opcodes 14 through 20 in
`primitives.rs:17`. Note the numbering gap: 7, 8, and 9 are unassigned, so the wire is not densely packed.

Per-op payload limits, each cited:

```
  MAX_INPUT_BYTES        = 65536      hash inputs (BLAKE3/SHA-256/384/512/SHA3-256)   types.rs:40
  MAX_VERIFY_MESSAGE     = 1 MiB      Ed25519 message                                types.rs:38
  MAX_AEAD_PT_BYTES      = 1 MiB      AEAD plaintext (seal) / ciphertext (open)      types.rs:41
  MAX_AEAD_AAD_BYTES     = 256        AEAD associated data                           types.rs:42
  AEAD key / nonce / tag = 32 / 12 / 16                                              types.rs:43
  HMAC_KEY_MAX           = 256        HMAC key                                       primitives.rs:28
  HKDF_PART_MAX          = 256        each of HKDF salt / ikm / info                 primitives.rs:29
  HKDF_OUT_MAX           = 512        HKDF output length (out_len in 1..=512)        primitives.rs:30
  X25519_KEY_BYTES       = 32         X25519 scalar / point                          primitives.rs:25
  P256_VERIFY_BYTES      = 65+64+32 = 161   exact P-256 verify payload               primitives.rs:26
  P384_VERIFY_BYTES      = 97+96+48 = 241   exact P-384 verify payload               primitives.rs:27
  MAX_PAYLOAD_BYTES      = max(AEAD path, verify path) + header cushion              types.rs:50
```

The shared envelope budget `MAX_PAYLOAD_BYTES` is the larger of the AEAD plaintext path
(`AEAD_HEADER_BYTES + MAX_AEAD_AAD_BYTES + MAX_AEAD_PT_BYTES + AEAD_TAG_BYTES`) and the verify path
(`ED25519_HEADER_BYTES + MAX_VERIFY_MESSAGE_BYTES`), so the receive buffer is sized once for the worst
case (`types.rs:50`). A request over that budget is rejected in the decoder as `BadLength` before a
handler runs (`decode.rs:43`); a per-op oversize inside the budget is `EMSGSIZE`; a malformed frame is
`EINVAL`.

The three ops with a non-trivial internal frame:

The AEAD frame is shared between AES-256-GCM and ChaCha20-Poly1305 and parsed once in
`handlers/aead_frame/`. The common header is `key[32] || nonce[12] || u32 aad_len`, then `aad_len` bytes
of associated data, then the body (`aead_frame/common.rs:20`, `aead_frame/constants.rs:21`). For a seal
the body is the plaintext, bounded by `MAX_PT` (`aead_frame/parse.rs:21`); for an open the body is
`ciphertext || tag`, which must be at least `TAG_LEN` and at most `MAX_PT + TAG_LEN`
(`aead_frame/parse.rs:37`). A body over the limit is `OversizePayload` (mapped to `EMSGSIZE`); a short
frame or an over-large `aad_len` is `Short`/`OversizeAad` (mapped to `EINVAL`)
(`aes256_gcm_seal.rs:27`). Seal additionally rejects an all-zero nonce as degenerate before encrypting
(`aead_frame/parse.rs:29`, `aes256_gcm_seal.rs:41`), a real guard against the worst AES-GCM misuse since a
repeated nonce under the same key is catastrophic for GCM. Open verifies the tag and returns the
plaintext or `EBADMSG`; a tampered ciphertext or wrong AAD is rejected rather than returned
(`aes256_gcm_open.rs:44`).

Ed25519 verify takes `pubkey[32] || sig[64] || message` and returns a status only, never
attacker-influenced bytes (`ed25519_verify.rs:38`). An undersize payload or an over-1-MiB message is
`EMSGSIZE`, a malformed public key is `EINVAL`, a signature that does not check is `EBADMSG`, and a good
one is status `0` (`ed25519_verify.rs:39`). Because the reply carries no body, the op cannot be used to
smuggle data out. The two ECDSA verifies (P-256, P-384) are the same status-only shape but require an
exact payload length and use SEC1-encoded keys with prehashed digests (`p256_ecdsa_verify.rs:23`,
`p384_ecdsa_verify.rs:23`).

HKDF-SHA256 is a hand-written HKDF over the capsule's own HMAC-SHA256. The payload is `u16 out_len || u16
salt_len || u16 ikm_len || u16 info_len`, then those three byte runs (`hkdf_sha256.rs:40`). The parse
requires the declared lengths to exactly consume the payload (`end != payload.len()` is rejected), so a
length-field mismatch cannot over-read (`hkdf_sha256.rs:49`). Extract is `PRK = HMAC(salt, ikm)`, and
Expand iterates `T(i) = HMAC(PRK, T(i-1) || info || counter)` for `ceil(out_len / 32)` blocks and
truncates, the standard RFC 5869 construction (`hkdf_sha256.rs:35`, `hkdf_sha256.rs:55`). HMAC-SHA256
itself is the textbook ipad/opad construction over SHA-256, hand-written in `handlers/hmac_core.rs:22`.

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` initializes the heap and, on success, enters `server::run`;
a heap-init failure exits with code 1 (`src/main.rs:28`). There are two top-level modules: `protocol`
(the wire) and `server` (the loop and handlers) (`src/main.rs:22`).

The `protocol` module is the frame: `decode_request` validates and slices an incoming buffer
(`protocol/decode.rs:27`), `encode_response` builds the reply header plus status plus body
(`protocol/encode.rs:21`), `errno` holds the four status codes (`protocol/errno.rs`), and `types.rs` plus
`primitives.rs` hold the magic, version, opcodes, and every size constant. The public surface is
re-exported through `protocol/mod.rs:23`.

The `server` module is the loop and the handler tree. `dispatch` routes one op to one handler
(`server/dispatch.rs:28`); `handlers/` holds one file per primitive, re-exported through
`handlers/mod.rs:37`; `wipe` is the volatile buffer zero (`server/wipe.rs:19`); and `runner` is the loop
(`server/runner.rs:29`).

Lifecycle:

1. The kernel spawns the capsule at boot through `spawn_plan/core.rs:57`, which calls
   `boot::capsule("CRYPTO", "crypto", spawn_crypto_capsule, shared_state)`. `spawn_crypto_capsule` decodes
   the baked trust anchor, builds a `CapsuleSpecVerified` from the embedded ELF, cert, manifest, and
   attestation trailer, requests `IPC | Memory | Crypto`, and calls `spawn_verified`, which verifies the
   whole chain before the code runs (`src/security/crypto_capsule/spawn.rs:40`).
2. On success the boot path registers the capsule with lifecycle and logs `[CRYPTO] capsule spawned`
   (`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). The registry then
   resolves `crypto_pool` on port 4102 for callers.
3. `run` allocates one receive buffer of `HDR_LEN + MAX_PAYLOAD_BYTES` and loops: `mk_ipc_recv` on inbox
   0, `decode_request`, `dispatch`, `mk_ipc_send` to the reply endpoint `0x1_0000_0004`, then `wipe` over
   exactly the `n` received bytes (`server/runner.rs:29`).
4. The wipe writes each byte through `core::ptr::write_volatile` and issues a `SeqCst` compiler fence, so
   the compiler cannot elide the zeroing of key and plaintext bytes from the receive buffer
   (`server/wipe.rs:19`).

## Protocol and IPC

The capsule is a pure server: it receives on inbox 0 and replies to the fixed kernel reply endpoint
`0x1_0000_0004` (`server/runner.rs:32`, `server/runner.rs:41`). It makes no outbound calls of its own and
speaks only the NOCX frame above. Every operation carries the same header; a request supplies `op` plus a
per-op payload, and the reply is `header || i32 status || body`.

The callers reach it through the layered path described in the overview: userland capsule ->
`nonos_libc::crypto_*` shim -> crypto syscall -> kernel crypto-capsule client -> IPC to `crypto_pool`.
The kernel client exposes fifteen of the seventeen ops (`src/security/crypto_capsule/client/mod.rs:37`):
the four hashes with in-tree clients (BLAKE3, SHA-256, SHA-512, SHA3-256), Ed25519 verify, both AEAD
pairs, HMAC and HKDF, both X25519 ops, and healthcheck. The two ECDSA verifies (P-256, P-384) and
SHA-384 exist in the capsule but have no in-tree kernel client yet, which is noted as a gap below.

The kernel client gates each op with `CAP_CRYPTO` at the top of the request path: `gate_hash` reads the
caller pid from process accounting and checks `has_capability(pid, CAP_CRYPTO)` before marshalling
(`src/security/crypto_capsule/capability.rs:22`). It is called at the top of the hash path
(`client/hash_op.rs:33`), both AEAD paths (`client/aead_op/seal.rs:34`, `client/aead_op/open.rs:34`), the
PRF path that carries HMAC, HKDF, and both X25519 ops (`client/prf_op.rs:37`), the Ed25519 verify path
(`client/verify_ed25519.rs:48`), and healthcheck (`client/healthcheck.rs:24`). So the capability check is
per request, on the caller pid, on the kernel side.

Concrete callers in the tree, each reaching the pool through the `nonos_libc` crypto shim
(`userland/libc/src/crypto/mod.rs:30`):

- The [market](../market/README.md) verifies its signed catalog index through `crypto_ed25519_verify` rather than
  linking Ed25519 itself (`userland/capsule_market/src/verify/crypto.rs:30`).
- The keyring builds EIP-712 digests and Ethereum addresses through the hash and keccak shims
  (`userland/capsule_keyring/src/server/eip712/digest.rs`,
  `userland/capsule_keyring/src/server/handlers/sign_receipt/sign_receipt.rs`).
- The Nym transport uses the AEAD, ECDH, hash, and KDF shims for its onion crypto
  (`userland/capsule_net_nym/src/crypto/aead.rs:17`, `net_nym/src/crypto/ecdh.rs`,
  `net_nym/src/crypto/kdf.rs:20`).
- The ramfs store seals and opens its at-rest records through the AEAD shims
  (`userland/capsule_ramfs/src/store/crypto/seal.rs`, `ramfs/src/store/crypto/open.rs`).
- The wallet's TLS 1.3 client drives its handshake through the hash, HKDF, and AEAD record shims
  (`userland/capsule_wallet_nonos/src/wallet/tls13/hkdf.rs`,
  `wallet_nonos/src/wallet/tls13/record_seal.rs`).

## Security analysis

The pool centralizes crypto so that a caller needs neither a key nor a crypto crate of its own, and the
security model rests on that being a real reduction in privilege rather than a new attack surface.

The gate is per request and lives on the kernel side, not in the capsule. Every op the kernel client
exposes runs `gate_hash` first, which fails closed with `AccessDenied` if the caller pid lacks
`CAP_CRYPTO` (`capability.rs:27`). This corrects a natural misreading: it is not the per-op payload limits
that stand in for a capability check. The payload limits are a separate defense (they bound the heap a
single request can consume), but the authority decision is a genuine capability test on the caller pid
before the request is even sent. Within the capsule there is no per-caller check because the capsule never
sees caller authority by design (`Capsule.mk:1`); the kernel has already decided.

The capsule's own mask `0x39` is `CoreExec | IPC | Memory | Crypto` and nothing else
(`Capsule.mk:16`, `spawn.rs:54`). The concentration of the RustCrypto crates here buys no extra authority:
the mask holds no `FileSystem`, so a compromised crypto crate cannot write to a storage surface; no
`Network`, so it cannot exfiltrate a key it was handed to seal with; no `Driver`, `Mmio`, `Irq`, `Dma`, or
`Pio`, so it cannot reach hardware; no `Debug`, so it cannot log the plaintext it processes. Its isolation
is statelessness: no key or session survives a request (`server/runner.rs:29`), so there is nothing to
leak between callers, and the reply endpoint `0x1_0000_0004` is distinct from ramfs, keyring, and entropy
so in-flight replies cannot cross-route (`types.rs:60`).

Constant-time posture is inherited, not hand-rolled. Verification and AEAD tag checks come from
`ed25519-dalek`, `aes-gcm`, `chacha20poly1305`, `p256`, and `p384` (`Cargo.toml:28`), which carry their
own constant-time guarantees; the capsule adds status-only replies for the verify ops so a failing check
returns a boolean-shaped `EBADMSG` with no body (`ed25519_verify.rs:63`). The SHA-2 crate is pinned to
`force-soft` (`Cargo.toml:25`), a portable software path. The one place the capsule reasons about misuse
itself is the degenerate-nonce guard on AEAD seal (`aead_frame/parse.rs:29`).

Handling of secret bytes is careful but not perfect. The whole receive buffer is volatile-zeroed after
every reply (`server/runner.rs:42`, `server/wipe.rs:19`), and `x25519_shared` additionally wipes its
copied-out private scalar before returning (`x25519_shared.rs:35`). The crypto crates zeroize their own
internal key state (the `zeroize` features are enabled for the dalek crates, `Cargo.toml:31`).

Honest gaps:

- The intermediate parsed key bytes inside a handler (the slice handed to a cipher) are not separately
  zeroized beyond the whole-buffer wipe, except for the explicit `x25519_shared` scalar wipe; they rely on
  the receive-buffer wipe reaching them.
- There is no rate limiting, so a flood of 1 MiB Ed25519 verifies or AEAD opens is not throttled.
- The P-256, P-384, and SHA-384 ops are implemented in the capsule but have no in-tree kernel client
  (`src/security/crypto_capsule/client/mod.rs:37`), so no `CAP_CRYPTO`-gated caller reaches them yet.
- The capsule mixes RustCrypto crates (`aes-gcm`, `chacha20poly1305`, `ed25519-dalek`, `x25519-dalek`,
  `p256`, `p384`, `sha2`, `sha3`, `blake3`) with hand-written HMAC and HKDF (`hmac_core.rs`,
  `hkdf_sha256.rs`) rather than being a single implementation, and it is a separate body of code from the
  kernel's in-tree [crypto stack](../../subsystems/crypto/README.md).

## How to contribute

The source lives at `userland/capsule_crypto/`. The wire is under `src/protocol/`, the loop and handlers
under `src/server/`, one file per primitive under `src/server/handlers/`.

To add a primitive:

1. Assign an opcode. If it fits the original block, add the constant to `src/protocol/types.rs:20`;
   otherwise add it to `src/protocol/primitives.rs:17` alongside the later ops, and any size constant
   next to it. Re-export the new names through `src/protocol/mod.rs:23` or `:34`.
2. Write the handler as one file under `src/server/handlers/`, exposing `pub fn name(req: Request<'_>) ->
   Vec<u8>` that validates its payload, computes, and returns `encode_response(op, req.flags,
   req.request_id, status, body)`. Follow the existing shape: bound the input against a named constant and
   return `EMSGSIZE`/`EINVAL` on a bad frame (for example `sha256_hash.rs:23`), and for a verify op return
   a status with an empty body (`ed25519_verify.rs:69`).
3. Declare the module and re-export the handler in `src/server/handlers/mod.rs:17` and `:37`.
4. Wire the opcode into the match in `src/server/dispatch.rs:29`.
5. If a kernel-side caller should reach it, add a client under `src/security/crypto_capsule/client/`,
   re-export it in `client/mod.rs:37`, call `gate_hash()?` first (`capability.rs:22`), and add the syscall
   plumbing under `src/syscall/dispatch/crypto/` plus the `nonos_libc` shim under
   `userland/libc/src/crypto/`.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_crypto/Capsule.mk:19`):

```
  make nonos-mk-crypto              build the capsule ELF
  make nonos-mk-crypto-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-crypto-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-crypto-keys   check the per-capsule signing keys exist
```

For a running image that includes the pool, `make nonos-mk-crypto-prod` builds the crypto-profile kernel
(`Makefile:920`), and `make nonos-mk-boot-crypto-hash` runs the hash round-trip boot harness
(`Makefile:1390`, `tests/boot/crypto_hash_round_trip.sh`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns errors as a status code, never a panic; the release
profile is `panic = "abort"`, `Cargo.toml:37`); modular files, one primitive per file, with `mod.rs` used
only for re-exports; and the AGPL header at the top of every source file, matching the header on every
existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints `[CRYPTO]
capsule spawned` (tag `CRYPTO`, message `capsule spawned`) from the boot log
(`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line means the
capsule never started, usually a signature, manifest, or capability failure; the error path prints an
`[ERROR]` line with the mapped `SpawnError` instead (`capsule_boot/run.rs:32`). A present marker means
`crypto_pool` resolves on port 4102, so callers such as the [market](../market/README.md) can reach it.

Because the pool is stateless, its request-time failure signatures map cleanly to cause
(`src/protocol/errno.rs:17`):

- `EMSGSIZE` (-90) is an oversize input against a per-op cap: 64 KiB for the hashes, 1 MiB for an AEAD
  body or an Ed25519 message, or an out-of-range HKDF `out_len` (`sha256_hash.rs:24`,
  `ed25519_verify.rs:43`, `hkdf_sha256.rs:29`).
- `EINVAL` (-22) is a malformed frame: a short AEAD header, an over-large `aad_len`, a malformed Ed25519
  public key, an X25519 key of the wrong length, a wrong-length ECDSA payload, an HKDF length-field
  mismatch, or a degenerate all-zero AES-GCM nonce (`aead_frame/parse.rs`, `ed25519_verify.rs:49`,
  `x25519_public.rs:23`, `hkdf_sha256.rs:49`, `aes256_gcm_seal.rs:41`). An unknown opcode is also `EINVAL`
  (`dispatch.rs:47`), as is any frame the decoder rejects for bad magic, version, or length
  (`runner.rs:39`).
- `EBADMSG` (-74) is a verification that did not pass: an Ed25519 or ECDSA signature that did not check,
  or an AEAD open whose tag did not authenticate (`ed25519_verify.rs:65`, `aes256_gcm_open.rs:46`).
- `EIO` (-5) is an AEAD seal failure returned by the cipher (`aes256_gcm_seal.rs:49`).

The one worth recognizing is `EBADMSG` on a verify: because verification returns a status and no body, a
failing verify is a clean boolean-shaped answer, so a market install that reports a key rejection traces
back through the `ED25519_VERIFY` op returning `EBADMSG` on the index signature
(`userland/capsule_market/src/verify/crypto.rs:37`). If a caller sees `AccessDenied` instead of any wire
status, the failure is upstream of the capsule: the kernel client's `gate_hash` rejected the caller pid
for lacking `CAP_CRYPTO`, and no frame was ever sent (`capability.rs:27`).

## Source map

```
  userland/capsule_crypto/src/main.rs                     _start -> heap_init -> server::run
  userland/capsule_crypto/src/protocol/types.rs           NOCX frame, ops 1..13, size constants
  userland/capsule_crypto/src/protocol/primitives.rs      ops 14..20 and their size constants
  userland/capsule_crypto/src/protocol/{decode,encode}.rs frame decode and reply encode
  userland/capsule_crypto/src/protocol/errno.rs           EIO EINVAL EBADMSG EMSGSIZE
  userland/capsule_crypto/src/server/runner.rs            the recv/dispatch/send/wipe loop
  userland/capsule_crypto/src/server/dispatch.rs          op -> handler match
  userland/capsule_crypto/src/server/wipe.rs              the volatile buffer wipe
  userland/capsule_crypto/src/server/handlers/            one file per primitive (hash/aead/verify/kdf/ecdh)
  userland/capsule_crypto/src/server/handlers/aead_frame/ shared AEAD frame parse + degenerate-nonce guard
  userland/capsule_crypto/src/server/handlers/hmac_core.rs, hkdf_sha256.rs   hand-written HMAC / HKDF
  userland/capsule_crypto/Capsule.mk                      slug, handle, ports, capability mask, kernel mirror
  src/security/crypto_capsule/                            the kernel-side embed, verified spawn, and client
  src/security/crypto_capsule/capability.rs               gate_hash: per-op CAP_CRYPTO on the caller pid
  src/security/crypto_capsule/client/                     the fifteen kernel clients + transport
  src/userspace/init/spawn_plan/core.rs                   the boot spawn entry
  userland/libc/src/crypto/                               the nonos_libc crypto shims callers use
  nonos-mk/capsule.mk                                     the generated nonos-mk-crypto[-sign|-verify] targets
```

Every reference above is verified against those trees.

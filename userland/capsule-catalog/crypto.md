# capsule_crypto

`capsule_crypto` is the userland cryptographic compute pool: a stateless service that takes a hash,
signature-verify, AEAD, KDF, or ECDH request, computes it, replies, and wipes the request buffer. It
holds no keys and no session state. It also serves a second purpose that is worth stating up front: it is
where the userland's cryptographic-crate dependencies are *isolated*, so that other capsules reach AEAD
and Ed25519 through IPC rather than each linking a crypto library. Service `crypto_pool`, reply endpoint
`0x1_0000_0004`, capability mask `0x39`. The source is `userland/capsule_crypto/`.

## Contents

- [The server loop](#the-server-loop)
- [The wire protocol](#the-wire-protocol)
- [Payload caps](#payload-caps)
- [The primitive families](#the-primitive-families)
- [AEAD in detail](#aead-in-detail)
- [Ed25519 verification in detail](#ed25519-verification-in-detail)
- [HKDF in detail](#hkdf-in-detail)
- [Provenance: where these primitives come from](#provenance-where-these-primitives-come-from)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop

`main.rs:28` initializes the heap and calls `server::run` (`src/server/runner.rs:29`):

```
  run():
      loop:
          n = mk_ipc_recv(inbox 0, buf)
          req  = decode_request(buf[..n])       // magic 0x4E4F4358 "NOCX", version 1; else EINVAL
          resp = dispatch(req)                    // stateless, no store
          mk_ipc_send(KERNEL_REPLY_ENDPOINT, resp)
          wipe(buf[..n])                           // volatile-zero every received byte
```

The buffer is volatile-zeroed after every reply, so the key and plaintext bytes carried in a seal or open
request do not remain in the receive buffer. The reply endpoint `0x1_0000_0004` is deliberately distinct
from ramfs, keyring, and entropy so concurrent in-flight requests to different pools cannot cross-route.

## The wire protocol

The frame is a 20-byte header (`src/protocol/types.rs:64`), the same shape as
[capsule_entropy](entropy.md): `u32 magic, u16 version, u16 op, u16 flags, u16 reserved, u32 request_id,
u32 payload_len`. The operations (`src/protocol/types.rs:20`):

```
  1  BLAKE3_HASH       6  ED25519_VERIFY        14 X25519_PUBLIC     18 P256_ECDSA_VERIFY
  2  SHA3_256_HASH    10  CHACHA20_POLY1305_SEAL 15 X25519_SHARED    19 P384_ECDSA_VERIFY
  3  HEALTHCHECK      11  CHACHA20_POLY1305_OPEN 16 HMAC_SHA256      20 SHA384_HASH
  4  SHA256_HASH      12  AES256_GCM_SEAL        17 HKDF_SHA256
  5  SHA512_HASH      13  AES256_GCM_OPEN
```

`dispatch` (`src/server/dispatch.rs`) routes each to a handler; an unknown op is `EINVAL`.

## Payload caps

Each handler enforces its own bound, and the envelope budget is derived from the largest
(`src/protocol/types.rs:40`):

```
  MAX_INPUT_BYTES      = 64 KiB      // hash inputs
  MAX_AEAD_PT_BYTES    = 1 MiB       // AEAD plaintext
  MAX_AEAD_AAD_BYTES   = 256 B       // AEAD associated data
  MAX_VERIFY_MESSAGE   = 1 MiB       // Ed25519 message
  AEAD key/nonce/tag   = 32 / 12 / 16
  MAX_PAYLOAD_BYTES    = max(AEAD path, verify path) + header cushion
```

An oversize request is `EMSGSIZE`; a malformed one is `EINVAL`. The caps stop a hostile caller from
parking a multi-megabyte buffer in the crypto capsule's heap through one request.

## The primitive families

The pool covers five families: **hashes** (BLAKE3, SHA-256/384/512, SHA3-256), **AEAD** (AES-256-GCM and
ChaCha20-Poly1305, seal and open), **signature verification** (Ed25519, P-256 and P-384 ECDSA), **KDF and
MAC** (HKDF-SHA256, HMAC-SHA256), and **ECDH** (X25519 public and shared). All are pure functions of the
request, so a request never depends on a prior one.

## AEAD in detail

`aes256_gcm_seal` (`src/server/handlers/aes256_gcm_seal.rs:25`) parses an AEAD frame, guards against nonce
misuse, and encrypts:

```
  aes256_gcm_seal(req):
      frame = parse_seal(payload)        // key[32] || nonce[12] || _[4] || aad || plaintext
              Short | OversizeAad -> EINVAL ;  OversizePayload -> EMSGSIZE
      if nonce_is_degenerate(frame.nonce):  return EINVAL      // reject an all-zero nonce
      cipher = Aes256Gcm::new(frame.key)
      ct = cipher.encrypt(nonce, { msg: plaintext, aad })
      return ct   (or EIO on failure)
```

The `nonce_is_degenerate` check rejects an all-zero nonce, a small but real guard against the worst
AES-GCM misuse (a repeated nonce under the same key is catastrophic for GCM). The `open` variant verifies
the tag and returns the plaintext or fails; a tampered ciphertext or wrong associated data is rejected
rather than returned. The ChaCha20-Poly1305 pair is analogous.

## Ed25519 verification in detail

`ed25519_verify` (`src/server/handlers/ed25519_verify.rs:38`) takes `pubkey[32] || sig[64] || message`
and returns a **status only**, never attacker-influenced bytes:

```
  ed25519_verify(req):
      if payload < 96:                    EMSGSIZE
      if message_len > 1 MiB:             EMSGSIZE
      key = VerifyingKey::from_bytes(pubkey)      else EINVAL     // malformed public key
      sig = Signature::from_bytes(sig)
      match key.verify(message, sig):
          Ok  -> status 0
          Err -> status EBADMSG
```

A malformed public key is `EINVAL`, an oversize message is `EMSGSIZE`, and a signature that does not check
is `EBADMSG`. Because the response is a status and no body, the operation cannot be used to smuggle data.
The [market](market.md) verifies its signed index through this op rather than linking Ed25519 itself.

## HKDF in detail

`hkdf_sha256` (`src/server/handlers/hkdf_sha256.rs:24`) is a hand-written HKDF over the capsule's own
HMAC-SHA256:

```
  hkdf_sha256(req):
      (out_len, salt, ikm, info) = parse(payload)    // u16 out_len, u16 salt/ikm/info lens, then bytes
      bounds: out_len in 1..=HKDF_OUT_MAX ; each part <= HKDF_PART_MAX
      prk = HMAC-SHA256(salt, ikm)                    // Extract
      out = Expand(prk, info, out_len)                // T(i) = HMAC(prk, T(i-1) || info || i)
      return out
```

The `expand` function (`hkdf_sha256.rs:55`) is the standard RFC 5869 HKDF-Expand: it iterates
`T(i) = HMAC-SHA256(prk, T(i-1) || info || counter)` for `ceil(out_len / 32)` blocks and truncates to the
requested length. The parse validates that the declared part lengths exactly consume the payload
(`end != payload.len()` is rejected), so a length-field mismatch cannot over-read.

## Provenance: where these primitives come from

This is the honest, precise part. `capsule_crypto` does not use the kernel's in-tree
[crypto stack](../../subsystems/crypto/README.md); it uses a mix, and it is worth being exact:

- **AES-256-GCM** is the RustCrypto `aes-gcm` crate (`use aes_gcm::Aes256Gcm`), and **Ed25519** is
  `ed25519-dalek` (`use ed25519_dalek::VerifyingKey`). The capsule's own source comments this: crypto
  lives here so that a caller like the market reaches Ed25519 through the kernel transport rather than
  taking a direct dependency. So this capsule *isolates* the crypto-crate dependencies in one place.
- **HKDF and HMAC** are hand-written in the capsule (`hkdf_sha256.rs`, `hmac_core.rs`) over SHA-256.
- The **kernel's** in-tree AES-GCM, ChaCha20-Poly1305, and Ed25519 (documented in the
  [crypto subsystem](../../subsystems/crypto/README.md)) are a separate implementation used inside the
  kernel; the AEAD and X25519 ops here also reach kernel primitives through `nonos_libc` syscalls where
  applicable. The point for a reader is that "the crypto capsule" and "the kernel crypto stack" are two
  distinct bodies of code, and this page documents the capsule.

## Security analysis

- **Statelessness**: no key or session survives a request, so there is nothing to leak between callers.
- **Per-op caps** bound the heap a single request can consume.
- **Buffer wipe** after every reply erases key and plaintext bytes from the receive buffer.
- **The degenerate-nonce guard** blocks the most dangerous AES-GCM misuse.
- **Status-only verification** means a verify op returns a boolean-shaped result, not data.

## Honest gaps

Stated plainly: while the receive buffer is wiped, the intermediate parsed key bytes inside a handler
(the slice handed to the cipher) are not separately zeroized, so they rely on the buffer wipe; the crypto
libraries zero their own internal state. There is no rate limiting, so a flood of 1 MiB Ed25519 verifies
is not throttled. And, as above, the capsule mixes RustCrypto crates with hand-written and kernel
primitives rather than being a single implementation.

## Source map

```
  userland/capsule_crypto/src/server/runner.rs           the loop + wipe
  userland/capsule_crypto/src/server/dispatch.rs          op -> handler
  userland/capsule_crypto/src/server/handlers/aead_frame.rs   parse_seal, nonce_is_degenerate
  userland/capsule_crypto/src/server/handlers/aes256_gcm_seal.rs, chacha20_*    the AEAD ops (aes-gcm crate)
  userland/capsule_crypto/src/server/handlers/ed25519_verify.rs                 (ed25519-dalek crate)
  userland/capsule_crypto/src/server/handlers/hkdf_sha256.rs, hmac_core.rs      hand-written HKDF/HMAC
  userland/capsule_crypto/src/server/wipe.rs              the volatile buffer wipe
  userland/capsule_crypto/src/protocol/types.rs           the NOCX frame, ops, caps
```

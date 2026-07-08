# Hashes, MAC, and KDF

The kernel's most-used cryptographic primitives are its hashes: BLAKE3 keys the capability and
IPC MACs, Keccak-256 backs the Ethereum and syscall paths, and the SHA-2 family backs HMAC and
HKDF. This page documents the hashes, the keyed MAC and key-derivation built on them, and the
constant-time comparison that makes verification safe. The code is under `src/crypto/hash/` and
`src/crypto/util/`.

## BLAKE3

There are two BLAKE3 implementations in the tree, and both are used, so it is worth being precise
about which is which:

- The in-tree implementation, `src/crypto/hash/blake3/`, is a full no_std BLAKE3 (its own
  `chunk`, `compress`, `hasher`, `output` submodules) exposing `blake3_hash`, `blake3_keyed_hash`,
  `blake3_derive_key`, and a `Hasher`. It is re-exported through `crate::crypto`. The
  [capability token MAC](../../security/signing-and-mac.md) uses it directly, via
  `blake3_keyed_hash` (`src/capabilities/token/material.rs:42`).
- The external `blake3` crate (a pinned dependency in `Cargo.toml`) is used where code writes a
  bare `blake3::Hasher`. The [IPC message MAC](../ipc/envelope.md) is the notable case
  (`src/ipc/nonos_channel/hash.rs`): it calls `blake3::Hasher::new_keyed` and `new_derive_key`
  with no `use crate::crypto`, so the path resolves to the crate.

Both are real BLAKE3; the in-tree one exists so the primitive is available without the crate on
paths that want it, and the crate is used where its builder API is convenient. The in-tree
implementation is checked against the official BLAKE3 test vectors
(`userland/crypto_proofs/src/blake3_tests.rs`), covering the plain, keyed, derive-key, and XOF
modes.

## SHA-2 and Keccak

The SHA-2 family and Keccak are in-tree no_std implementations, each with a known-answer test:

```
  SHA-256      src/crypto/hash/unified/sha256.rs   NIST FIPS 180-4 vectors
  SHA-512      src/crypto/hash/sha512/             NIST FIPS 180-4 vectors
  SHA-384      src/crypto/hash/sha384.rs           SHA-512 engine with the SHA-384 IV
  Keccak-256   src/crypto/hash/sha3/keccak.rs      SHA-3 vectors
```

SHA-256 and SHA-512 are hand-written compression functions; SHA-384 reuses the SHA-512 engine
with the standard alternate initialization vector. Keccak-256 is a full in-tree Keccak sponge,
and it is the one the Ethereum address derivation, the secp256k1 ECDSA hashing, and the
`MkCryptoKeccak256` syscall all use. The KAT files live in `userland/crypto_proofs/`, which
compiles the real primitive source and runs it against published vectors, so the in-tree hashes
are proven against the standards rather than asserted to match them.

## HMAC and HKDF

HMAC and HKDF are in-tree, built over the SHA-2 hashes (`src/crypto/util/hmac/`):
`hmac_sha256` and `hmac_sha512` are the standard HMAC construction, and `hkdf_extract` /
`hkdf_expand` are HKDF over HMAC. They back the `MkCryptoHmacSha256` and `MkCryptoHkdfSha256`
syscalls and the capsule key-derivation paths, and each has a KAT
(`userland/crypto_proofs/src/hmac_tests.rs`, `hkdf_tests.rs`). HMAC verification compares the
recomputed tag in constant time.

## Constant-time comparison

Any comparison of a secret or a MAC goes through the constant-time helpers
(`src/crypto/util/constant_time/`), `ct_eq` and its fixed-width forms, which fold the difference
across the whole input rather than returning at the first mismatch. This is what keeps MAC and
signature verification from leaking, through timing, where a forged value first diverges. The
capability MAC check, the HMAC verify, and the field-element comparisons inside the signature
code all use it. There is a constant-time test in `userland/crypto_proofs/src/constant_time_tests.rs`.

## Source

```
  src/crypto/hash/blake3/            the in-tree BLAKE3 (capability MAC path)
  src/crypto/hash/unified/sha256.rs  SHA-256
  src/crypto/hash/sha512/            SHA-512 and SHA-384
  src/crypto/hash/sha3/keccak.rs     Keccak-256
  src/crypto/util/hmac/              HMAC and HKDF
  src/crypto/util/constant_time/     constant-time comparison
  userland/crypto_proofs/src/        the known-answer tests
```

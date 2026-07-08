# Asymmetric Cryptography

The kernel implements the classical public-key primitives in-tree: Ed25519 for signatures,
secp256k1 for ECDSA, and the elliptic-curve field arithmetic underneath both. Ed25519 is the
signature scheme the kernel itself signs with and the capsule trust chain verifies with. This
page documents them and the kernel's own signing key. The code is under
`src/crypto/asymmetric/` and `src/crypto/kernel_keys.rs`.

## Ed25519

Ed25519 (`src/crypto/asymmetric/ed25519/`) is a full in-tree implementation, the Edwards-curve
field, point, scalar, and signature math written for no_std, checked against the RFC 8032 test
vectors including adversarial edge cases (`userland/crypto_proofs/src/ed25519_tests.rs`). It is
the kernel's primary signature primitive:

- The kernel holds one Ed25519 keypair (`kernel_keys.rs`), generated once at init behind a `Once`,
  and signs capability tokens with it (`sign_capability_token`). The public half is exported so a
  token can be checked against it.
- The [capsule trust chain](../../security/capsules-and-trust.md) verifies Ed25519 signatures on
  the NØNOS-ID certificate and the manifest.
- The `MkCryptoEd25519Verify` syscall exposes verification to capsules.

## secp256k1

secp256k1 (`src/crypto/asymmetric/secp256k1/`) is an in-tree implementation of the Bitcoin and
Ethereum curve, point arithmetic in affine and projective coordinates plus ECDSA signing and
public-key recovery, with a KAT (`userland/crypto_proofs/src/secp256k1_tests.rs`). Its consumers
are the Ethereum transaction signing and address paths and the `MkCryptoSecp256k1Sign` and
`MkCryptoSecp256k1Pubkey` syscalls, where it hashes with the in-tree [Keccak-256](hashes.md). The
field arithmetic under both curves is the in-tree `src/crypto/util/bigint/`.

## The algorithm-id dispatch

Signature verification is generic over an algorithm id (`src/crypto/asymmetric/alg_id/`): the
`AlgId` enum names `Ed25519` and `MlDsa65`, and `verify` dispatches a `(pubkey, msg, sig)` to the
right primitive. This is the mechanism the [certificate and manifest policy](pqc.md) builds on to
require both algorithms at once: the low-level dispatch verifies one signature of a stated
algorithm, and the production policy requires a valid signature under each of the two.

## x25519 and the honest caveat

x25519 (`src/crypto/asymmetric/curve25519/`) is present but is **not** on the kernel trusted path.
It is feature-gated: with `crypto-curve25519` it binds to the `x25519_dalek` crate, and without
the feature the in-tree fallback is incomplete. It is used only by legacy network code paths, not
by capsule spawn, IPC, or the capability system. The `MkCryptoX25519*` syscalls are correspondingly
feature-gated. This page records that honestly rather than presenting x25519 as a load-bearing
kernel primitive.

Other asymmetric primitives exist in the proof tree (P-256, P-384, RSA test vectors under
`userland/crypto_proofs/`), but the kernel trusted path relies on Ed25519 and secp256k1 among the
classical schemes, and ML-DSA-65 among the post-quantum ones.

## Source

```
  src/crypto/asymmetric/ed25519/       Ed25519 (kernel signing, trust chain, syscall)
  src/crypto/asymmetric/secp256k1/     secp256k1 ECDSA (Ethereum, syscall)
  src/crypto/asymmetric/alg_id/        the AlgId verify dispatch
  src/crypto/asymmetric/curve25519/    x25519 (feature-gated, not trusted path)
  src/crypto/kernel_keys.rs            the kernel's Ed25519 keypair
```

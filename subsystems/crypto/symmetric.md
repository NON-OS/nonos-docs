# Symmetric Encryption

The kernel carries two authenticated encryption schemes, AES-256-GCM and ChaCha20-Poly1305,
both in-tree no_std implementations, exposed to capsules through the crypto syscall family. This
page documents them and the AEAD surface. The code is under `src/crypto/symmetric/` and
`src/crypto/core/`.

## The ciphers

Both AEAD constructions are implemented in the tree, not pulled from a crate:

```
  AES / AES-256-GCM        src/crypto/symmetric/aes/, aes_gcm/
                           in-tree AES (S-boxes, key schedule, CTR) + GHASH
  ChaCha20-Poly1305        src/crypto/symmetric/chacha20poly1305/
                           in-tree ChaCha20 stream + Poly1305 MAC
```

AES-256-GCM is the block cipher in counter mode with the GHASH Galois authenticator;
ChaCha20-Poly1305 is the stream cipher with the Poly1305 one-time authenticator. Both are
authenticated encryption with associated data (AEAD): encryption produces ciphertext plus an
authentication tag, and decryption fails if the tag does not verify, so a tampered ciphertext or
associated-data mismatch is rejected rather than returned. Each has a known-answer test that runs
the real source against published vectors (`userland/crypto_proofs/src/aesgcm_tests.rs`,
`chacha_tests.rs`).

## The AEAD core

The two schemes are unified behind an AEAD abstraction (`src/crypto/core/aead.rs`): `Aes256GcmAead`
and `Chacha20Poly1305Aead` implement the same trait, so callers select a scheme without duplicating
the seal-and-open logic. The core exposes encrypt (seal) and decrypt (open) with and without
associated data, and the tag handling is internal, so a caller cannot accidentally accept
unauthenticated plaintext.

## The crypto syscall family

Capsules reach these primitives through the `MkCrypto*` syscalls, which the router dispatches to
`crypto::dispatch_crypto` (see the [syscall router](../syscall/router.md)). The family covers the
symmetric, hash, and asymmetric operations from one place (`src/syscall/dispatch/crypto/`):

```
  MkCryptoEncrypt / MkCryptoDecrypt   AES-256-GCM or ChaCha20-Poly1305 (caller-selected)
  MkCryptoHash / MkCryptoKeccak256    SHA-256, Keccak-256
  MkCryptoHmacSha256 / HkdfSha256     MAC and KDF
  MkCryptoEd25519Verify               signature verification
  MkCryptoSecp256k1Sign / Pubkey      ECDSA sign and public-key recovery
  MkCryptoX25519Public / Shared       ECDH (feature-gated)
  MkCryptoRandom                      secure random bytes
```

The encrypt and decrypt calls validate and copy the user buffers through the
[usercopy](../memory/usercopy.md) boundary before touching them, and the AEAD tag check means a
capsule that submits a corrupted ciphertext gets a failure, not garbage plaintext. The syscall
layer is a thin dispatch over the same in-tree primitives documented here; it adds the user
boundary and the capability check, not new crypto.

## Source

```
  src/crypto/symmetric/aes/, aes_gcm/       AES-256-GCM
  src/crypto/symmetric/chacha20poly1305/    ChaCha20-Poly1305
  src/crypto/core/aead.rs                    the AEAD trait and the two impls
  src/syscall/dispatch/crypto/               the MkCrypto* dispatch
  userland/crypto_proofs/src/                the AEAD known-answer tests
```

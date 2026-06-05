# Crypto

The cryptography is in-tree, `no_std`, and organised by purpose. This page lists
what is present in `src/crypto` and, more importantly, what each primitive is
actually used for: capsule admission, capability tokens, and the chain-facing
application layer are three different jobs with three different primitive sets.
The [architecture overview](../architecture/overview.md) summarises this in
section 15.

---

## The modules

```
  src/crypto/
    asymmetric/   ed25519, secp256k1, p256, p384, rsa, curve25519
    pqc/          ml_dsa_65, kyber, sphincs, ntru, mceliece
    hash/         blake3, sha3, sha512
    symmetric/    aes, aes_gcm, chacha20poly1305
    zk/           groth16 (over bn254), halo2
    util/         constant_time, hmac, entropy, rng
    kernel_keys   the token signing key and related material
```

## What signs a capsule

Capsule admission is the critical path. A capsule is signed twice over the same
material, once classical and once post-quantum, and both must verify
([capsules and trust](../security/capsules-and-trust.md)):

```
  algorithm    id      public key    signature     role
  ---------    ----    ----------    ----------    ----
  Ed25519      0x01    32 bytes      64 bytes      classical signature
  ML-DSA-65    0x03    1952 bytes    3309 bytes    post-quantum signature (FIPS 204)
```

Verification dispatches on the algorithm id
(`src/crypto/asymmetric/alg_id/verify.rs:23`): Ed25519 through the in-tree ed25519
code, ML-DSA-65 through the post-quantum module. Requiring both means a future
break of one scheme does not by itself forge a capsule.

## What hashes and authenticates

```
  BLAKE3      NONOS-ID derivation:  BLAKE3(handle || domain || recovery)
              capsule payload hash: BLAKE3(elf) checked against the manifest
              capability token MAC: two keyed BLAKE3 hashes, 64 bytes
```

BLAKE3 carries three jobs in the trust path. It derives the capsule identity in
the certificate, it hashes the ELF so the manifest binds to the exact code, and
keyed, it is the MAC that authenticates every capability token
([capabilities and tokens](../security/capabilities-and-tokens.md)). The MAC is
`keyed_blake3(key, material)` concatenated with `keyed_blake3(key, material ||
"CAP2")`, and it is compared in constant time so verification leaks no timing.

SHA3 and SHA-512 are present in the hash module for general use; the capsule trust
path itself uses BLAKE3.

## What faces the chain

The application layer, not the boot or spawn trust path, uses the
Ethereum-compatible primitives:

```
  secp256k1     signing and public-key recovery (Ethereum signatures)
  Keccak256     Ethereum-style hashing
  Groth16/BN254 zero-knowledge proof verification
  Halo2         an alternative proof system, feature gated
```

BN254 is the same pairing-friendly curve used by Ethereum's alt_bn128
precompiles, so a Groth16 proof verified here is verifiable in that ecosystem and
the other way around. These are exposed for capsules that do chain-facing or
privacy-preserving work; they are not part of admitting a capsule or minting a
token.

## What encrypts

```
  AES-GCM             authenticated symmetric encryption
  ChaCha20-Poly1305   authenticated symmetric encryption
```

Both are authenticated (AEAD), so a ciphertext that was tampered with fails to
decrypt rather than returning garbage. They are used where data needs
confidentiality and integrity together.

## The post-quantum set

Beyond ML-DSA-65, the `pqc` module carries Kyber (key encapsulation), SPHINCS+
(hash-based signatures), and NTRU and McEliece (alternative KEMs). ML-DSA-65 is
the one wired into the capsule signature path today; the rest are available for
key exchange and for hedging signature choices as the post-quantum landscape
settles.

---

## Exposed to capsules

A capsule does not carry its own crypto. The kernel exposes its primitives as
syscalls (the `Crypto*` family in the [ABI reference](../abi/syscalls.md)), so a
capsule asks the kernel to hash, encrypt, derive a key, or verify a signature
rather than linking a second implementation. Random bytes come from the kernel
CSPRNG (`util/rng`, seeded from `util/entropy`), so capsules share one audited
entropy source rather than each rolling its own.

## The split worth remembering

```
  admission + tokens   Ed25519, ML-DSA-65, BLAKE3        critical path, in-kernel
  chain-facing         secp256k1, Keccak256, Groth16/BN254   application layer
  data at rest/in flight  AES-GCM, ChaCha20-Poly1305          as needed
```

Keep these three apart when reasoning about the system. A weakness in the
chain-facing set does not touch capsule admission, and the admission set is
deliberately small, classical plus post-quantum signatures and one hash, so the
trust path is easy to audit.

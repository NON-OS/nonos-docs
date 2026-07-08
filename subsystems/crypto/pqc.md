# Post-Quantum Cryptography

NØNOS is post-quantum in its trust chain: a capsule's certificate and manifest must carry a valid
ML-DSA-65 signature in addition to an Ed25519 one, so an adversary who breaks elliptic-curve
signatures still cannot forge a capsule. This page documents the post-quantum primitives and where
the dual-signature requirement is enforced. The code is under `src/crypto/pqc/`.

## The primitives

The post-quantum primitives are FFI wrappers over the PQClean reference C implementations, not
in-tree Rust:

```
  ML-DSA-65 (Dilithium)   src/crypto/pqc/ml_dsa_65/   FFI to PQCLEAN_MLDSA65_CLEAN_*
  Kyber / ML-KEM          src/crypto/pqc/kyber.rs      FFI to PQCLEAN_MLKEM{512,768,1024}_CLEAN
```

ML-DSA-65 is the NIST post-quantum signature standard (the Dilithium submission), and Kyber /
ML-KEM is the post-quantum key-encapsulation standard. Both are thin Rust surfaces
(`ml_dsa_65_keypair` / `sign` / `verify`, `kyber_keygen` / `encaps` / `decaps`) calling into
PQClean, whose known-answer vectors live in the C library rather than in the tree. This is called
out plainly: unlike the [hashes](hashes.md) and [classical signatures](asymmetric.md), which are
in-tree with KATs under `userland/crypto_proofs/`, the PQC primitives are external reference code
reached by FFI.

## The dual-signature requirement

The place the post-quantum signature becomes load-bearing is the capsule production policy. The
NØNOS-ID certificate policy (`src/security/nonos_id_cert/policy.rs:32`) states it directly:

```
  // nonos-production policy: hybrid Ed25519 + ML-DSA-65, both required.
  SignaturePolicy { required: &[AlgId::Ed25519, AlgId::MlDsa65] }
```

Under `NONOS_PRODUCTION_POLICY`, a certificate or manifest must carry a valid signature under
*both* Ed25519 and ML-DSA-65, verified through the [algorithm-id dispatch](asymmetric.md). The
spawn [preflight](../elf-loader/integration.md) runs the certificate and manifest verification
under this policy before an image is loaded, so a capsule that is missing either signature does not
spawn. The generic `AlgId::verify` checks one algorithm at a time; the policy is what turns that
into "both are required", and the production policy requires both.

Kyber / ML-KEM is available for post-quantum key encapsulation in hybrid schemes but is not part of
the capsule trust chain; the chain is a signature story, and the signature is the hybrid
Ed25519 + ML-DSA-65 pair.

## Source

```
  src/crypto/pqc/ml_dsa_65/api.rs, ffi.rs   ML-DSA-65 over PQClean
  src/crypto/pqc/kyber.rs                    Kyber / ML-KEM over PQClean
  src/crypto/asymmetric/alg_id/verify.rs     the per-algorithm verify dispatch
  src/security/nonos_id_cert/policy.rs       NONOS_PRODUCTION_POLICY: both signatures required
```

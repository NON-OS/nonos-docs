# Capsule Attestation

The signature chain in [verified spawn](capsules-and-trust.md) proves who signed a
capsule and that its bytes match a signed manifest. Attestation is a second,
independent layer on top of that: a zero-knowledge proof, carried in a trailer
appended to the capsule, that the capsule's enrolled secret is a member of a
committed policy tree, bound to the capsule's exact code, its granted
capabilities, and a policy epoch, without revealing the secret. This is the
transparent, trapdoor-free enrolled-secret proof. This page documents the gate,
its enforcement, the trailer format, and exactly what the proof binds.

## Where the gate runs

Attestation is the last step of preflight, after the certificate and the manifest
have both verified and the installable capability set has been computed
(`preflight.rs:62`). The gate is `attest_gate`
(`src/kernel_core/process_spawn/capsule_spawn/runner/attest_gate.rs:20`). It takes
the capsule spec and the `install_caps` that verified spawn just computed, so the
proof is checked against the capabilities the capsule is actually about to
receive, not against what it requested.

## Enforcement is feature-gated

Whether a failed or absent attestation blocks a spawn depends on the
`nonos-zk-rollout` build feature, and the documentation states this exactly
because it is a real difference in behaviour between builds.

If the capsule carries no attestation trailer, the gate logs `[ZK-ATTEST] none`
and then, in a build without `nonos-zk-rollout`, returns
`SpawnError::AttestationRejected`; in a build with the feature, it returns `Ok`
(`attest_gate.rs:27`). If the trailer is present but verification fails, the gate
logs `[ZK-ATTEST] FAIL` with the reason and again rejects in a strict build and
returns `Ok` in a rollout build (`attest_gate.rs:47`). In other words, the rollout
feature is a soft-launch mode: attestation is parsed, verified, and logged, but a
missing or failing proof does not stop the capsule. A strict build, without the
feature, makes a valid attestation mandatory for every capsule.

## What the proof binds

When a trailer is present the gate calls `verify_capsule_attestation`, whose
return is marked `#[must_use]` with the note that a capsule must not be spawned
unless its attestation verifies (`src/security/capsule_attest/verify.rs:24`):

```
  verify_capsule_attestation(trailer, elf, granted_caps):
      proof = parse(trailer)?
      root = policy_root::root().ok_or(RootUnavailable)?
      capsule_hash = blake3(elf)                       32 bytes
      ctx[0..32]  = capsule_hash
      ctx[32..40] = granted_caps      (big-endian u64)
      ctx[40..48] = POLICY_EPOCH      (big-endian u64)
      if verify_enrolled(proof, root, ctx) -> Ok else -> Rejected
```

The context the proof is checked against is a 48-byte value that ties the proof to
three things at once: the exact ELF, via its BLAKE3 hash; the capability set being
installed, so a proof valid for one grant is not valid for a wider one; and the
policy epoch, so a proof enrolled under an old policy does not verify under a new
one. `verify_enrolled` (`src/crypto/zk_kernel/`) checks the enrolled-secret proof
against the committed policy root over that context; the zero-knowledge machinery
itself is documented in the crypto section. The policy root comes from
`policy_root::root()`, and if it is unavailable the gate rejects with
`RootUnavailable` rather than skipping the check.

## The trailer format

The trailer is a fixed-magic, fixed-layout blob parsed by `parse`
(`src/security/capsule_attest/trailer.rs:28`). It begins with the eight-byte magic
`NZKCAPS2`; a blob shorter than eight bytes or with the wrong magic is rejected as
`Missing`. The parser then requires an exact length,
`137 + depth*32 + ceil(depth/8)` where `depth` is `POLICY_TREE_DEPTH`, and it
requires the depth byte at offset 136 to equal that constant, rejecting anything
else as `Malformed`. The fixed prefix is:

```
  0..8     magic "NZKCAPS2"
  8..40    commitment       the Pedersen commitment to the enrolled secret
  40..72   nonce_point      the proof's nonce commitment
  72..104  z_x              the response for the secret
  104..136 z_r              the response for the blinding
  136      depth            must equal POLICY_TREE_DEPTH
  137..    siblings         depth entries of 32 bytes, the Merkle path
  then     directions       ceil(depth/8) bytes, one bit per level
```

The parsed result is an `EnrolledSecretProof`
(`src/crypto/zk_kernel`) carrying the commitment, the nonce point, the two
responses `z_x` and `z_r`, and a Merkle inclusion path of `siblings` and
per-level `directions` bits. The commitment, nonce point, and responses are a
Sigma-protocol proof of knowledge of the enrolled secret behind the commitment;
the Merkle path proves that commitment sits in the policy tree whose root the
kernel holds. Together they prove membership in policy without revealing which
member.

## Errors

The attestation error type is closed (`src/security/capsule_attest/error.rs:18`):

```
  Missing          "capsule attestation trailer missing"      bad or absent magic
  Malformed        "capsule attestation trailer malformed"    wrong length or depth
  RootUnavailable  "capsule attestation policy root unavailable"
  Rejected         "capsule attestation rejected"             verify_enrolled failed
```

The gate maps any of these to `SpawnError::AttestationRejected` in a strict build.

## Relationship to the signature chain

Attestation and the signature chain answer different questions and neither
subsumes the other. The certificate and manifest signatures prove that a known
publisher signed this exact capsule and that its declared capabilities are within
what the trust anchor allows. The attestation proves that the capsule is an
enrolled member of a policy tree, bound to its code and its granted capabilities,
in zero knowledge. A capsule can be correctly signed but not enrolled, or enrolled
under a stale policy epoch, and the attestation layer is what catches that,
independently of the signatures, when a strict build requires it.

This page covers the gate, the trailer, and what the proof binds. The cryptographic
construction underneath, the transparent Pedersen commitment with its
nothing-up-my-sleeve generator, the Schnorr-style membership proof, and the honest
classical-versus-post-quantum boundary, is documented on the
[proof system](../subsystems/proof-system/pedersen-attestation.md) page, alongside the
in-kernel transparent [STARK](../subsystems/proof-system/README.md).

## Source

```
  src/kernel_core/process_spawn/capsule_spawn/runner/attest_gate.rs  the gate and feature flags
  src/security/capsule_attest/verify.rs   verify_capsule_attestation and the context
  src/security/capsule_attest/trailer.rs  the NZKCAPS2 trailer format
  src/security/capsule_attest/layout.rs   POLICY_TREE_DEPTH, POLICY_EPOCH
  src/security/capsule_attest/policy_root.rs  the committed policy root
  src/security/capsule_attest/error.rs    AttestError
  src/crypto/zk_kernel/                     verify_enrolled and EnrolledSecretProof
```

# Capsule Attestation

The signature chain in [verified spawn](capsules-and-trust.md) proves who signed a
capsule and that its bytes match a signed manifest. Attestation is a second,
independent layer on top of that: a proof, carried in a trailer appended to the
capsule, that the capsule is a member of a committed policy tree, bound to the
capsule's exact code, its granted capabilities, and a policy epoch.

Two proof backends sit behind the same gate, selected at build time by the
`nonos-stark-attest` feature. The production build (`nonos-mk-zerostate`) verifies
a transparent, post-quantum FRI-STARK proof that the capsule's measurement is a
leaf of the policy tree (`src/security/capsule_attest/stark.rs:53`). Without the
feature, the default build verifies the earlier transparent, trapdoor-free
enrolled-secret proof (`verify_enrolled`, `src/crypto/zk_kernel/`). Both are
checked against the same 48-byte context and the same policy root; they differ
only in the proof system and the trailer layout. This page documents the gate,
both backends, and exactly what the proof binds. The STARK construction is on the
[proof system](../subsystems/proof-system/stark.md) page; the enrolled-secret
construction is on the
[Pedersen attestation](../subsystems/proof-system/pedersen-attestation.md) page.

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
      capsule_hash = blake3(elf)                       32 bytes
      ctx[0..32]  = capsule_hash
      ctx[32..40] = granted_caps      (big-endian u64)
      ctx[40..48] = POLICY_EPOCH      (big-endian u64)
      with nonos-stark-attest:  verify_capsule_attestation_stark(trailer, elf, granted_caps)
      without it:               parse(trailer); verify_enrolled(proof, root, ctx)
```

The context the proof is checked against is a 48-byte value that ties the proof to
three things at once: the exact ELF, via its BLAKE3 hash; the capability set being
installed, so a proof valid for one grant is not valid for a wider one; and the
policy epoch, so a proof enrolled under an old policy does not verify under a new
one. Both backends bind this same context. The STARK backend
(`src/security/capsule_attest/stark.rs:53`) reconstructs it, reads the policy root
from `policy_root::root()`, and verifies the money-grade membership proof against
that root. The enrolled-secret backend (`verify_enrolled`, `src/crypto/zk_kernel/`)
checks the Sigma-protocol proof against the same root. If the root is unavailable
either backend rejects with `RootUnavailable` rather than skipping the check.

## The enrolled-secret trailer format

This section describes the trailer of the enrolled-secret backend, magic
`NZKCAPS2`. The STARK backend uses its own trailer, magic `NZKSTRK1` (the
membership path followed by the serialized FRI-STARK proof), parsed in
`src/security/capsule_attest/stark.rs:53` and documented on the
[STARK](../subsystems/proof-system/stark.md) page.

The enrolled-secret trailer is a fixed-magic, fixed-layout blob parsed by `parse`
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

## Kernel self-attestation

The same membership proof runs one layer up, for the kernel itself. The bootloader
already measures the kernel image with BLAKE3; with the `stark-kernel-attest`
feature it also verifies the kernel's own STARK trailer, carried in the image
footer, against an enrolled kernel root before it jumps
(`nonos-bootloader/src/kernel_verify/stark_attest.rs:44`). The context is the kernel
measurement and the boot epoch. A zeroed root trusts nothing, so an un-enrolled
build cannot be spoofed into trusting a proof. Capsules attest to the kernel's
policy root; the kernel attests to its own enrolled measurement, and both are
checked by the same verifier, the `nonos-stark` crate the kernel and the
bootloader both link, so the prover and the verifier cannot drift.

## Relationship to the signature chain

Attestation and the signature chain answer different questions and neither
subsumes the other. The certificate and manifest signatures prove that a known
publisher signed this exact capsule and that its declared capabilities are within
what the trust anchor allows. The attestation proves that the capsule is an
enrolled member of a policy tree, bound to its code and its granted capabilities,
in zero knowledge. A capsule can be correctly signed but not enrolled, or enrolled
under a stale policy epoch, and the attestation layer is what catches that,
independently of the signatures, when a strict build requires it.

This page covers the gate, the trailers, and what the proof binds. The
cryptographic constructions underneath are documented on the proof system pages:
the production [STARK](../subsystems/proof-system/stark.md), a transparent,
post-quantum FRI proof with no trusted setup, and the earlier enrolled-secret
[Pedersen attestation](../subsystems/proof-system/pedersen-attestation.md), a
transparent but classical Sigma-protocol proof.

## Debugging attestation

An attestation failure surfaces at two places, and the first thing to do is read
which one. The gate itself prints one of three lines with the capsule name
(`attest_gate.rs:24`, `:35`, `:42`): `[ZK-ATTEST] none` means the capsule carried
no trailer, `[ZK-ATTEST] ok` means the proof verified, and `[ZK-ATTEST] FAIL`
means a trailer was present but did not verify. On a `FAIL` the gate appends the
`AttestError` string from `as_str` (`error.rs:26`), so the line reads, for
example, `[ZK-ATTEST] FAIL <name>: capsule attestation rejected`. That suffix is
the whole diagnosis: `trailer missing` for a bad or absent `NZKCAPS2` magic,
`trailer malformed` for a wrong length or depth byte, `policy root unavailable`
when `policy_root::root()` returned nothing, and `rejected` when the trailer
parsed and the group check in `verify_enrolled` (`attest/verify.rs:24`, which
returns `false`) did not pass.

The distinction between `none` and `FAIL` matters when a capsule will not spawn.
In a strict build (without `nonos-zk-rollout`) both a `none` and a `FAIL` become
`SpawnError::AttestationRejected` (`attest_gate.rs:28`, `:49`), and the capsule
loader turns that into `[RUNTIME-LOAD] FAILED name=<name> reason=attestation`
(`from_vfs/load.rs:98`). So a runtime-load failure with `reason=attestation` is
always this gate, never a signature problem: if the reason were a bad signature it
would read `reason=id_cert` or `reason=manifest:pub_sig` instead, and if it were a
capability overreach it would read `reason=manifest:caps_ceiling` or
`reason=manifest:grant`. The `reason=` field is the fastest way to separate an
attestation reject from a signature reject from a capability reject.

A `rejected` on a capsule that was enrolled correctly is usually a binding
mismatch rather than a forged secret. The 48-byte context ties the proof to the
ELF hash, the installed cap bitmask, and `POLICY_EPOCH`, so rebuilding the capsule
(new BLAKE3 over the ELF), changing the granted caps, or moving the policy epoch
all invalidate a proof that previously verified. The way to tell that apart from a
genuinely absent enrolled secret is whether the capsule bytes or its grant changed
since the proof was produced. A live read of the boot-chain result is available
through the `MkAttestStatus` syscall (`src/syscall/microkernel/attest.rs`), which
any valid token can call.

One honest caveat belongs here. A `[ZK-ATTEST] FAIL` or `none` in a build with
`nonos-zk-rollout` is logged and then ignored: the gate returns `Ok` and the
capsule spawns anyway. That rollout feature is mutually exclusive with
`nonos-production` (`src/lib.rs:39`, a `compile_error!`), so a production build
cannot be built fail-open, but during a rollout window a failing attestation is
not why a spawn fails, and this marker should be read as advisory, not as a gate.

## Source map

```
  src/kernel_core/process_spawn/capsule_spawn/runner/attest_gate.rs  the gate, the markers, and the feature flags
  src/kernel_core/process_spawn/capsule_spawn/from_vfs/load.rs       the [RUNTIME-LOAD] reason= mapping
  src/security/capsule_attest/verify.rs   verify_capsule_attestation, the backend dispatch and 48-byte context
  src/security/capsule_attest/stark.rs    the STARK backend: NZKSTRK1 parse and money-grade verify
  src/security/capsule_attest/trailer.rs  the NZKCAPS2 enrolled-secret trailer format
  src/security/capsule_attest/layout.rs   POLICY_TREE_DEPTH, POLICY_EPOCH
  src/security/capsule_attest/policy_root.rs  the committed policy root
  src/security/capsule_attest/error.rs    AttestError and its as_str messages
  src/crypto/zk_kernel/attest/verify.rs   verify_enrolled, the enrolled-secret constant-time group check
  nonos-bootloader/src/kernel_verify/stark_attest.rs  the kernel self-attestation the bootloader checks
  nonos-stark/                            the shared verifier crate, linked by kernel and bootloader
  src/lib.rs                              the nonos-production / nonos-zk-rollout exclusivity
```

The proof constructions verified above are on the
[STARK](../subsystems/proof-system/stark.md) and
[Pedersen attestation](../subsystems/proof-system/pedersen-attestation.md) pages;
the signature chain that produces the other `reason=` values is on the
[verified spawn](capsules-and-trust.md) page.

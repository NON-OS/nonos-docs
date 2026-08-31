# Capsule Attestation

The signature chain in [verified spawn](capsules-and-trust.md) proves who signed a
capsule and that its bytes match a signed manifest. Attestation is a second,
independent layer on top of that: a proof, carried in a trailer appended to the
capsule, that the capsule is a member of a committed policy tree, bound to the
capsule's exact code, its granted capabilities, and a policy epoch.

Two proof backends sit behind the same gate, selected at build time by the
`nonos-stark-attest` feature. The production build (`nonos-mk-zerostate`) verifies
a transparent, post-quantum FRI-STARK proof that the capsule's measurement is a
leaf of the policy tree (`src/security/capsule_attest/stark.rs:55`, `verify_against`).
Without the feature, the default build verifies the earlier transparent,
trapdoor-free enrolled-secret proof (`verify_enrolled`, `src/crypto/zk_kernel/`).
Both are checked against the same 48-byte context and against the vendor policy
root first, then any developer roots enrolled on this machine; they differ only in
the proof system and the trailer layout. This page documents the gate,
both backends, and exactly what the proof binds. The STARK construction is on the
[proof system](../subsystems/proof-system/stark.md) page; the enrolled-secret
construction is on the
[Pedersen attestation](../subsystems/proof-system/pedersen-attestation.md) page.

## Where the gate runs

Attestation is the last step of preflight, after the certificate and the manifest
have both verified. Preflight dispatches on the capsule's tier
(`preflight.rs:67`, `super::tier::classify(namespace)`): a `Tier::Enrolled` capsule
goes through `attest_gate` and a `Tier::Publisher` capsule through `publisher_gate`.
Both are passed the manifest's `required_caps` (`preflight.rs:65`, `:68`, `:70`),
not the full installed set, so the proof binds the capabilities the manifest
requires rather than any optional bits a spawn site chose to add.

The main gate is `attest_gate`
(`src/kernel_core/process_spawn/capsule_spawn/runner/attest_gate.rs:23`), signature
`fn attest_gate(spec, required_caps: u64) -> Result<Option<Proved>, SpawnError>`. On
success it returns `Some(Proved)`, the measurement and the authority that vouched
for it (see [what the proof binds](#what-the-proof-binds)), so the caller records
both rather than a bare pass.

`publisher_gate` (`.../runner/publisher_gate.rs:20`) is the publisher-tier path. For
a `Tier::Publisher` capsule that carries no trailer it logs `[ZK-ATTEST] pub` and
returns `Ok(None)`, admitting it unattested (`publisher_gate.rs:30`); a
publisher-tier capsule that does carry a trailer is verified like any other. This is
a deliberate policy carve-out for the publisher tier, and it is why a fourth marker,
`pub`, exists alongside `none`, `ok`, and `FAIL`. The tier split lives in
`.../runner/tier.rs`.

## Enforcement is feature-gated

Whether a failed or absent attestation blocks a spawn depends on the
`nonos-zk-rollout` build feature, and the documentation states this exactly
because it is a real difference in behaviour between builds.

If the capsule carries no attestation trailer, the gate logs `[ZK-ATTEST] none`
(`attest_gate.rs:30`) and then, in a build without `nonos-zk-rollout`, returns
`Err(SpawnError::AttestationRejected)` (`attest_gate.rs:34`); in a build with the
feature, it returns `Ok(None)` (`attest_gate.rs:36`). If the trailer is present and
verifies, the gate logs `[ZK-ATTEST] ok <name> <authority>` and returns
`Ok(Some(proved))` in both builds (`attest_gate.rs:41`). If the trailer is present
but verification fails, it logs `[ZK-ATTEST] FAIL <name>: <reason>`
(`attest_gate.rs:52`) and again rejects in a strict build (`attest_gate.rs:59`) or
returns `Ok(None)` in a rollout build (`attest_gate.rs:62`). In other words, the
rollout feature is a soft-launch mode: attestation is parsed, verified, and logged,
but a missing or failing proof does not stop the capsule. A strict build, without
the feature, makes a valid attestation mandatory for every non-publisher-tier
capsule.

## What the proof binds

When a trailer is present the gate calls `verify_capsule_attestation`, whose
return is marked `#[must_use]` with the note that a capsule must not be spawned
unless its attestation verifies (`src/security/capsule_attest/verify.rs:33`). It
returns not a bool but a `Proved { measurement, authority }`, and it tries two
authorities in a fixed order (`verify.rs:38-53`):

```
  verify_capsule_attestation(trailer, elf, granted_caps) -> Result<Proved, AttestError>:
      vendor = policy_root::root()  or  Err(RootUnavailable)
      if against_root::verify(trailer, elf, granted_caps, vendor) ok:
          return Proved { measurement, authority: Vendor }
      for root in enrolled_roots():                    # developer roots, if any
          if against_root::verify(trailer, elf, granted_caps, root) ok:
              return Proved { measurement, authority: authority_for(root) }
      return Err(Rejected)
```

The vendor root is tried first and always, and only then the developer roots a
user enrolled on this machine, so a capsule that verifies under the shipped policy
is never attributed to a local key. The dual-root model, the `Authority` it
records, and the trusted-path enrolment are on the
[developer roots](developer-roots.md) page.

The actual proof check is `against_root::verify`
(`src/security/capsule_attest/against_root.rs:27`), which takes the root as a
parameter and is where the 48-byte binding context is built and the backend is
selected:

```
  against_root::verify(trailer, elf, granted_caps, root):
      capsule_hash = blake3(elf)                       32 bytes
      ctx[0..32]  = capsule_hash
      ctx[32..40] = granted_caps      (big-endian u64)
      ctx[40..48] = POLICY_EPOCH      (big-endian u64)      # POLICY_EPOCH = 1, layout.rs:17
      with nonos-stark-attest:  stark::verify_against(trailer, elf, granted_caps, root)
      without it:               verify_enrolled(parse(trailer), root, ctx)
```

The context the proof is checked against is a 48-byte value that ties the proof to
three things at once: the exact ELF, via its BLAKE3 hash; the capability set the
proof is bound to, which is the manifest's `required_caps` passed in as
`granted_caps`, so a proof enrolled for one required set is not valid for another;
and the policy epoch, so a proof enrolled under an old policy does not verify under
a new one (`against_root.rs:45-48`; the STARK backend rebuilds the identical context
in `stark.rs:84-87`). Both backends bind this same context. The vendor root itself
comes from `policy_root::root()` (`policy_root.rs:17`), which `include_bytes!`es
`nonos-data/trust/policy/zk_capsule_policy_root.bin` and returns `None`, hence
`RootUnavailable`, if it is not exactly 32 bytes rather than skipping the check.

## The enrolled-secret trailer format

This section describes the trailer of the enrolled-secret backend, magic
`NZKCAPS2`. The STARK backend (`src/security/capsule_attest/stark.rs`) uses its own
trailer and its own entry point `verify_against`, which reconstructs the 48-byte
context and calls `stark_verify_ext_blown_bound` from the `nonos-stark` crate with
the attestation parameters `N_QUERIES = 32`, `GRIND_BITS = 16`, and
`EXTRA_BLOWUP_BITS = 3` (`nonos-stark/src/attest_params.rs:41`, `:45`, `:48`). Its
membership proof and those parameters are documented on the
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

The attestation error type is closed (`src/security/capsule_attest/error.rs:18`),
four variants with fixed messages (`as_str`, `error.rs:26`):

```
  Missing          "capsule attestation trailer missing"      bad or absent magic
  Malformed        "capsule attestation trailer malformed"    wrong length or depth
  RootUnavailable  "capsule attestation policy root unavailable"   the vendor root file is not 32 bytes
  Rejected         "capsule attestation rejected"             no root (vendor or enrolled) verified the proof
```

The gate maps any of these to `SpawnError::AttestationRejected` in a strict build.
Note that `Rejected` now means the proof verified under neither the vendor root nor
any enrolled developer root, since `verify_capsule_attestation` tries them all
before returning it (`verify.rs:53`).

## Kernel self-attestation

The same membership proof runs one layer up, for the kernel itself. The in-kernel
entry point is `verify_kernel_self_attestation`
(`src/security/kernel_attest.rs:40`), marked `#[must_use]` with the note that the
boot chain must halt if the kernel does not self-attest. It measures the kernel
image with BLAKE3, builds a 40-byte context of that measurement plus the boot epoch
(`kernel_attest.rs:45-48`, `BOOT_EPOCH = 1`, `DEPTH = 8`), and calls
`verify_membership_trailer` from the `nonos-stark` crate with the same attestation
parameters the capsule path uses (`N_QUERIES`, `GRIND_BITS`, `EXTRA_BLOWUP_BITS`
from `attest_params.rs`). The bootloader runs the mirror check before it jumps: it
measures the image and verifies the kernel's own STARK trailer, carried in the
image footer, against an enrolled kernel root
(`nonos-bootloader/src/kernel_verify/stark_attest.rs`). Capsules attest to the
kernel's policy root; the kernel attests to its own enrolled measurement, and both
are checked by the same verifier, the `nonos-stark` crate the kernel and the
bootloader both link, so the prover and the verifier cannot drift. Note the context
sizes differ by design: the capsule context is 48 bytes because it also binds the
granted capabilities, while the kernel context is 40 bytes, measurement plus epoch,
because a kernel has no capability grant to bind.

The verdict gates the jump. The boot decision refuses a kernel whose
self-attestation does not verify, the same hard refuse the invalid-signature and
rollback paths use, under a signature-required mode
(`nonos-bootloader/src/boot/crypto/signature/verify.rs`,
`.../signature/error.rs`). A signature alone is not enough: the kernel must also
prove its measurement, or the boot resets rather than continue.

The enrolled root is mandatory at build time. When `stark-kernel-attest` is on,
`build.rs` fails the build if `NONOS_KERNEL_ATTEST_ROOT` is missing or all zeroes,
rather than embedding a zero root, so a build that turns the gate on cannot ship
one that silently trusts nothing (`nonos-bootloader/build.rs`,
`generate_kernel_attest_root`). Enroll the kernel first, then build with the gate.

## Runtime attestation registry

Boot measurements say what the machine loaded; they say nothing about a capsule
spawned an hour later. The registry is the runtime half
(`src/security/attest_registry/`). Every capsule that passes this gate is recorded
with the measurement its proof was checked against (`record_attested`), and the
entry is removed when the capsule exits (`forget_attested`). `registry_root` folds
the live set into one digest (`attest_registry/root.rs`), which is the value a
higher-level attestation over the running system would sign, and `attested_count`
reports how many are live. `registry_complete` (`attest_registry/complete.rs:29`)
is consulted before a [developer root](developer-roots.md) may be enrolled: a
machine that has already lost track of what it is running must not gain a new
authority whose future output its own attestation could not describe. The registry
is in-memory and does not survive a reboot, consistent with the rest of the
capability and enrolment state.

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
which one. The gate prints one of four lines with the capsule name:
`[ZK-ATTEST] none` (`attest_gate.rs:30`) means the capsule carried no trailer,
`[ZK-ATTEST] ok` (`attest_gate.rs:41`) means the proof verified and the line also
names the authority that vouched, `[ZK-ATTEST] FAIL` (`attest_gate.rs:52`) means a
trailer was present but did not verify, and `[ZK-ATTEST] pub`
(`publisher_gate.rs:30`) means a publisher-tier capsule with no trailer was admitted
unattested by the publisher carve-out. On a `FAIL` the gate appends the
`AttestError` string from `as_str` (`error.rs:26`), so the line reads, for
example, `[ZK-ATTEST] FAIL <name>: capsule attestation rejected`. That suffix is
the whole diagnosis: `trailer missing` for a bad or absent magic, `trailer
malformed` for a wrong length or depth byte, `policy root unavailable` when the
vendor root file was not 32 bytes, and `rejected` when the proof verified under
neither the vendor root nor any enrolled developer root (in the default build, the
group check in `verify_enrolled`, `zk_kernel/attest/verify.rs:24`, returned
`false`).

The distinction between `none` and `FAIL` matters when a capsule will not spawn.
In a strict build (without `nonos-zk-rollout`) both a `none` and a `FAIL` become
`SpawnError::AttestationRejected` (`attest_gate.rs:34`, `:59`). For a capsule
loaded from the store, that verdict is wrapped as `LoadError::Spawn(SpawnError::AttestationRejected)`
(`from_vfs/error.rs`, and `SpawnError` at `capsule_spawn/spec.rs:48`), which carries
the exact spawn verdict rather than a reason string. `AttestationRejected` is a
distinct `SpawnError` variant from `NonosIdCertRejected(IdCertVerifyError)` and
`ManifestRejected(ManifestVerifyError)`, so the variant itself separates an
attestation reject from a signature reject from a capability reject: a bad
certificate is `NonosIdCertRejected`, a bad publisher signature or a capability
overreach is `ManifestRejected(PublisherBadSig)` or
`ManifestRejected(CapsExceedCeiling / GrantOutsideManifest)`, and only this gate
produces `AttestationRejected`. The `[ZK-ATTEST]` console line is the fastest live
signal; the `SpawnError` variant is the exact one.

A `rejected` on a capsule that was enrolled correctly is usually a binding
mismatch rather than a forged secret. The 48-byte context ties the proof to the
ELF hash, the installed cap bitmask, and `POLICY_EPOCH`, so rebuilding the capsule
(new BLAKE3 over the ELF), changing the granted caps, or moving the policy epoch
all invalidate a proof that previously verified. The way to tell that apart from a
genuinely absent enrolled secret is whether the capsule bytes or its grant changed
since the proof was produced. A live read of the boot-chain result is available
through the `MkAttestStatus` syscall, `sys_attest_status`
(`src/syscall/microkernel/attest.rs:35`), which any valid token can call. It
surfaces the `AttestStatusAbi` (`attest.rs:26`) the bootloader recorded in the
handoff (`Measurements` plus `ZkAttestation`); it reports the recorded verdict, it
does not re-run the proof, so it is a status read and not a second verification.

One honest caveat belongs here. A `[ZK-ATTEST] FAIL` or `none` in a build with
`nonos-zk-rollout` is logged and then ignored: the gate returns `Ok` and the
capsule spawns anyway. That rollout feature is mutually exclusive with
`nonos-production` (`src/lib.rs:44`, a `compile_error!`), so a production build
cannot be built fail-open, but during a rollout window a failing attestation is
not why a spawn fails, and this marker should be read as advisory, not as a gate.
`src/lib.rs` carries three such guards in all: the arch-selection guard
(`lib.rs:32`), a guard forbidding development capsule flags in a production build
(`lib.rs:38`), and this rollout guard (`lib.rs:44`).

## Proving it under attack

The attestation is exercised adversarially by tooling that links the same
verifier the bootloader links, over the same image byte layout the boot path
parses, so a green run is evidence about the shipped gate rather than a model of
it (`security/nonos-secops`). Two tools sit behind it.

`nonos-defend` is the blue-team side: it parses an attested image footer and
verifies the kernel self-attestation against the enrolled root, the same check
the bootloader runs, and refuses to bless an image that would not boot.

`nonos-attack` is the red-team side. Its `battery` mode builds a genuine attested
image and then mounts the attacks a shipped image must survive, one per finding,
running each against the boot-side verify and passing only when the gate refuses
it: flip a byte in the flashed kernel, truncate the image, swap a foreign kernel
under a stolen trailer, forge a trailer under a different root, flip a bit inside
the trailer, and hand the parser an undersized image. Its `fuzz` mode drives the
untrusted trailer parser with random bytes and mutations of a real trailer,
including absurd length prefixes, and requires the parser to stay total: a parser
that can be driven to panic is a boot-time denial of service, so the fuzzer exits
non-zero if any input breaks it.

The suite runs on every pull request through the `attestation-attack` job in the
verify workflow, alongside the kernel self-attestation proof of concept
(`nonos-bootloader/tools/embed-zk-proof/tests/kernel_self_attest_poc.rs`), so a
change that weakens the gate or breaks the parser fails CI before it lands. Run it
locally with `security/tests/run.sh`.

## FAQ

Is the STARK real, or a stub that returns true?
Real. `nonos-stark` is a FRI-STARK with a DEEP quotient low-degree test, Merkle
openings, and a query phase. The verifier the tools link, the kernel links, and
the bootloader links is one crate, which is why an adversarial run of `nonos-attack`
is evidence about the gate that actually ships.

Is there a trusted setup?
No. There is no structured reference string, no ceremony, and no toxic waste. The
only public input is a hash. That is what transparent means here.

Is it actually post-quantum?
The attestation path uses a hash and a finite field and nothing else. There is no
discrete log, no pairing, and no factoring for a quantum adversary to attack. The
kernel signature layer is post-quantum too: ML-DSA-65 alongside Ed25519.

What is the soundness, exactly?
It is set by explicit parameters in `nonos-stark/src/attest_params.rs`:
`N_QUERIES = 32` FRI queries (`:41`), `GRIND_BITS = 16` transcript proof-of-work
bits (`:45`), `EXTRA_BLOWUP_BITS = 3` (`:48`), the attestation hash at `2^5` full
rounds (`LOG_ROUNDS = 5`, `:37`), and challenges drawn over the quadratic extension
`Fp2`. Those are real FRI soundness knobs and can be dialled up. The parameters,
not a vibe.

How do you revoke a capsule or a kernel?
Bump the epoch. `POLICY_EPOCH` (`layout.rs:17`, currently 1) and `BOOT_EPOCH`
(`kernel_attest.rs:33`, currently 1) are inside the bound context, so rotating an
epoch invalidates every trailer built under the old one. Revocation is a re-enroll
under the new epoch.

What records what is running after boot?
The [attestation registry](#runtime-attestation-registry)
(`src/security/attest_registry/`). Every capsule that passes this gate is recorded
with the measurement its proof was checked against and removed when it exits;
`registry_root` folds that set into one digest, and `registry_complete` gates
whether new developer roots may be enrolled.

What happens when attestation fails?
A capsule whose attestation does not verify is not spawned in a strict build; the
verifier return is `#[must_use]` so a spawn cannot ignore it. A kernel whose
self-attestation does not verify does not boot under a signature-required mode.

Does attestation replace the signatures?
No. It sits next to them. The kernel is dual-signed with its rollback index bound
and on top of that proves its measurement; the capsule is signed and on top of
that proves its measurement and its capability set. Defence in depth, not a swap.
See [relationship to the signature chain](#relationship-to-the-signature-chain).

Does turning the gate on break a build that has not enrolled yet?
Yes, deliberately. With `stark-kernel-attest` on, a build without a real enrolled
root fails rather than shipping a gate that trusts nothing. Enroll first.

## Source map

```
  src/kernel_core/process_spawn/capsule_spawn/runner/attest_gate.rs  the gate, the markers, and the feature flags
  src/kernel_core/process_spawn/capsule_spawn/runner/publisher_gate.rs  the [ZK-ATTEST] pub carve-out and tier::classify
  src/kernel_core/process_spawn/capsule_spawn/spec.rs               SpawnError, incl. AttestationRejected
  src/kernel_core/process_spawn/capsule_spawn/from_vfs/error.rs     LoadError { Manifest, TrustAnchor, Spawn(SpawnError) }
  src/security/capsule_attest/verify.rs   verify_capsule_attestation, the vendor-then-enrolled root order
  src/security/capsule_attest/against_root.rs  the 48-byte context and the backend dispatch
  src/security/capsule_attest/proved.rs   Proved { measurement, authority }
  src/security/capsule_attest/stark.rs    the STARK backend: verify_against and money-grade verify
  src/security/capsule_attest/trailer.rs  the NZKCAPS2 enrolled-secret trailer format
  src/security/capsule_attest/layout.rs   POLICY_TREE_DEPTH = 8, POLICY_EPOCH = 1
  src/security/capsule_attest/policy_root.rs  the committed vendor policy root
  src/security/capsule_attest/error.rs    AttestError and its as_str messages
  src/security/dev_roots/                  the enrolled developer roots and the Authority enum
  src/security/kernel_attest.rs            verify_kernel_self_attestation (in-kernel)
  src/security/attest_registry/            the runtime registry of what is running
  src/syscall/microkernel/attest.rs        sys_attest_status (MkAttestStatus)
  nonos-stark/src/attest_params.rs         N_QUERIES, GRIND_BITS, EXTRA_BLOWUP_BITS, LOG_ROUNDS
  src/crypto/zk_kernel/attest/verify.rs   verify_enrolled, the enrolled-secret constant-time group check
  nonos-bootloader/src/kernel_verify/stark_attest.rs  the kernel self-attestation the bootloader checks
  nonos-bootloader/src/boot/crypto/signature/verify.rs  the boot decision that gates the jump on the verdict
  nonos-bootloader/src/boot/crypto/signature/error.rs   the hard refuse on a failed self-attestation
  nonos-bootloader/build.rs               the mandatory enrolled root when the gate is on
  nonos-bootloader/tools/embed-zk-proof/  enroll the kernel and embed its self-attestation trailer
  nonos-stark/                            the shared verifier crate, linked by kernel and bootloader
  security/nonos-secops/                  the blue-team verifier and the red-team attack and fuzz suite
  security/tests/run.sh                   the local security suite harness
  src/lib.rs                              the nonos-production / nonos-zk-rollout exclusivity
```

The proof constructions verified above are on the
[STARK](../subsystems/proof-system/stark.md) and
[Pedersen attestation](../subsystems/proof-system/pedersen-attestation.md) pages;
the signature chain that produces the other `reason=` values is on the
[verified spawn](capsules-and-trust.md) page.

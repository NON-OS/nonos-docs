# Capsules and the Trust Anchor

Nothing runs in NØNOS until it has proven itself. A capsule is not a program
loaded off a disk and started; it is a signed artifact that the kernel verifies
against a baked-in root of trust before a single instruction of it executes. This
page is the admission gate. It describes what a capsule is as an artifact, the
trust anchor that roots every check, and the exact, ordered pipeline that decides
whether a capsule is allowed to become a process and with what authority.

This is the pipeline that runs before the [capability
model](capabilities-and-tokens.md) applies. Verified spawn decides what a capsule
is allowed to hold; the capability token enforces it on every syscall thereafter.
The two are halves of one guarantee.

## What a capsule is

A capsule is three artifacts produced at build time and embedded directly into
the kernel binary:

```
  ELF                   the executable, compiled for the capsule user target
  NØNOS-ID certificate  the publisher identity, signed by the trust anchor
  manifest              payload hash, required and optional capabilities,
                        namespace, target triple, endpoints, and the
                        publisher signatures over all of it
```

All three are compiled into the kernel image with `include_bytes!`. There is no
disk in the trust path and no loader that reads an image off a filesystem; the
system is RAM-resident, and the bytes that are verified are the bytes that run.
The manifest is `CapsuleManifest` (`src/security/capsule_manifest/schema/manifest.rs:29`):
a schema version, the id of the certificate it is bound to, the namespace, the
capsule version, the target triple, the BLAKE3 hash of the ELF payload, the
`required_caps` and `optional_caps` bitmasks, the declared endpoints, and the
publisher signatures.

## The trust anchor

Every check in this page ultimately reduces to a signature that verifies against
the trust anchor, so the anchor is where trust begins and the only thing the
kernel trusts a priori. It is a policy blob baked into the kernel at build time
(`src/security/nonos_trust_anchor/baked.rs:22`):

```
  pub const BAKED_TRUST_ANCHOR_POLICY: &[u8] =
      include_bytes!(".../nonos-data/trust/policy/nonos_trust_anchor.policy.bin");
```

The choice to make this a plain `&[u8]` rather than an `Option` is deliberate and
load-bearing, and the source says why: a missing policy file must break the
build, not be papered over with a `None` or an empty slice, because either would
let an unverified capsule path quietly take over at runtime. The kernel cannot be
built without a trust anchor. The policy itself carries the anchor's signing
keys, a policy epoch, and revocation lists for certificate serials, publisher
identities, and publisher key ids.

## The gate is preflight, then install

A capsule is spawned through `spawn_verified`
(`src/kernel_core/process_spawn/capsule_spawn/runner/verified.rs:26`). Its shape
is the whole security argument in three lines: preflight, and only if preflight
returns, install.

```
  spawn_verified(spec, trust_anchor, now_ms):
      preflighted = preflight::run(spec, trust_anchor, now_ms)?    the gate
      install(... caps_bits: preflighted.install_caps ...)         the load
```

The `?` is the gate. If preflight returns an error, `spawn_verified` returns it,
and `install` never runs: no process is created, no memory is allocated, no ELF
is loaded. Verification is not a check performed on a running process; it is a
precondition for the process existing at all. The comment above the function
states the other half of the rule plainly (`verified.rs:23`): the capabilities
installed on the process come from the verified manifest, never from the caps the
spawn site requested. The requested set is only an upper bound the spawn site is
willing to grant for optional capabilities, a point the capability math below
makes exact.

Preflight (`.../runner/preflight.rs:29`) runs two verifications in order. First
the certificate is verified against the trust anchor. Then the manifest is
verified against that certificate. Only after both pass does an attestation gate
run and preflight return the installable capability set.

## Stage one: the certificate against the anchor

`verify` in `src/security/nonos_id_cert/verify/mod.rs:28` establishes the
publisher's identity against the anchor before anything the publisher says is
trusted:

```
  verify(cert_bytes, policy, sig_policy, now_ms):
      cert = decode(cert_bytes)
      checks::run(cert, policy, now_ms)             epoch, revocation, validity
      signed = signed_region::compute(cert, cert_bytes)
      for alg in sig_policy.required:
          dispatch::run(alg, cert, signed, policy, now_ms)   anchor signature
      -> VerifiedNonosId { nonos_id, cert_serial, allowed_caps_ceiling }
```

`checks::run` (`nonos_id_cert/verify/checks.rs`) rejects a certificate whose
policy epoch is older than the anchor's current epoch, whose serial or publisher
identity appears in the anchor's revocation lists, or whose validity window does
not contain the supplied time. `signed_region::compute` recomputes the exact byte
range the signature covers from the certificate structure rather than trusting a
length in the input. Then, for each algorithm the signature policy requires, a
trust-anchor signature over that region must verify against an anchor key. The
successful result is a `VerifiedNonosId`, and its most important field for what
follows is `allowed_caps_ceiling`, the hard limit on capabilities that this
identity may ever hold.

## Stage two: the manifest against the certificate

`verify_with_publisher` in `src/security/capsule_manifest/verify/mod.rs:36` is the
core of the gate, and its ordering is exact and deliberate:

```
  1  decode(manifest_bytes)
  2  cert_binding::check      the manifest is bound to this exact certificate
  3  namespace::check         the namespace is within the certificate's globs
  4  caps::check_ceiling      required|optional is within allowed_caps_ceiling
  5  for each required alg:    a publisher signature over the signed region
       dispatch::run            verifies against a publisher key in the cert
  6  payload::check           BLAKE3(elf) equals manifest.payload_hash
  7  target_triple::check     the target triple matches this kernel's
  8  endpoint_drift::check    every endpoint the spawn site registers is declared
     caps::check_grant        compute the installed capability set
     capsule_id::derive       derive the deterministic capsule id
```

Step 2 binds the manifest to a specific certificate: the manifest carries a
32-byte `nonos_id_cert_id`, and `cert_binding::check`
(`capsule_manifest/verify/cert_binding.rs`) compares it against a hash of the
certificate being verified, so a manifest cannot be presented against a different
certificate than the one it names. Step 3 confirms the manifest's namespace falls
within one of the glob patterns the certificate authorises. Step 4 is the first
capability gate and is described in full below. Step 5 verifies, for each required
algorithm, a publisher signature over the recomputed signed region against a key
carried in the certificate. Step 6 hashes the actual ELF and compares it to the
manifest, so the manifest authenticates the exact payload. Steps 7 and 8 confirm
the capsule was built for this kernel's target and that it does not register any
IPC endpoint it did not declare in the manifest.

Step 6 is worth isolating because it is where the signatures reach the code. The
publisher signatures in step 5 authenticate the manifest structure, not the
binary. The binary is bound in only by step 6: `blake3_hash(payload)` must equal
`manifest.payload_hash` (`capsule_manifest/verify/payload.rs:22`). Authenticating
the manifest and then binding the payload to a hash inside the authenticated
manifest is what ties a signed statement to an exact set of bytes.

## The capability math

Two functions bound a capsule's authority, and both are small enough to state
exactly (`src/security/capsule_manifest/verify/caps.rs`).

The ceiling check (`caps.rs:20`) rejects a manifest that asks for more than its
certificate allows:

```
  union = required_caps | optional_caps
  if union & !allowed_caps_ceiling != 0 -> CapsExceedCeiling
```

A publisher cannot widen a capsule's authority by editing a manifest, because the
ceiling is fixed in the anchor-signed certificate and the union of everything the
manifest asks for must fit under it.

The grant check (`caps.rs:31`) computes what is actually installed:

```
  allowed = required_caps | optional_caps
  if granted_caps & !allowed != 0 -> GrantOutsideManifest
  install = required_caps | (optional_caps & granted_caps)
```

Read that result carefully, because it is the exact authority a capsule receives.
Required capabilities are always installed; the spawn site cannot withhold them.
Optional capabilities are installed only where the manifest declared them as
optional and the spawn site granted them. And the spawn site cannot grant
anything the manifest did not declare at all: a `granted` bit outside the
manifest's `required | optional` is rejected outright. The installed set is
bounded independently by three statements, the certificate ceiling, the manifest
declaration, and the spawn-site grant, and none of them can exceed the others.

## Both signatures, every time

The signature policy used for production is `NONOS_PRODUCTION_POLICY`, and both
the certificate verification and the manifest verification iterate over its
required algorithms (`preflight.rs:37`, `preflight.rs:54`). Production requires
two: Ed25519, a classical elliptic-curve signature, and ML-DSA-65, a
post-quantum lattice signature standardised as FIPS 204. Both must verify over
the same signed region. A capsule is admitted only if it carries a valid
signature under each, so a future break of either algorithm alone does not admit
an unverified capsule. The per-algorithm verification dispatches on the algorithm
id in `src/crypto/asymmetric/alg_id/verify.rs`.

## Why the order is the order

The sequence is not stylistic. The certificate is verified against the anchor
before anything it asserts is trusted, so the `allowed_caps_ceiling` that bounds
the capability checks is known genuine before those checks run. The manifest is
bound to its certificate before its namespace and capability claims are read, so
those claims are read from a manifest that provably belongs to the verified
certificate. The capability ceiling is checked before the expensive signature
verification, so a manifest that overreaches is rejected cheaply. The payload
hash is checked after the manifest signatures, so the bytes are bound by an
already-authenticated statement. Every step runs on inputs an earlier step has
already made trustworthy, and any step returning an error aborts the whole spawn.

## What install does on success

Only when preflight returns does `install` run
(`.../runner/install/install.rs`). It creates the process in the `Ready` state,
not `Running`, loads the ELF into the new address space, installs exactly the
capability bits preflight computed into the process control block and mints the
capability token over them, allocates the kernel and user stacks, builds the
initial user context, registers the capsule's declared endpoints, and adds the
pid to the tail of the run queue. The capsule does not run until the
[scheduler](../subsystems/scheduler/README.md) reaches it, and when it does, its every
syscall is checked against the token minted here. How that token is built and
enforced is the [capability model](capabilities-and-tokens.md); how the process
it now belongs to is structured is the [process
model](../subsystems/process/README.md).

## Source

```
  src/security/nonos_trust_anchor/baked.rs        the baked-in anchor policy
  src/security/nonos_id_cert/verify/mod.rs        certificate verification
  src/security/nonos_id_cert/verify/checks.rs     epoch, revocation, validity
  src/security/capsule_manifest/schema/manifest.rs the manifest structure
  src/security/capsule_manifest/verify/mod.rs     the ordered manifest pipeline
  src/security/capsule_manifest/verify/caps.rs    the ceiling and grant math
  src/security/capsule_manifest/verify/payload.rs the payload hash check
  src/crypto/asymmetric/alg_id/verify.rs          per-algorithm signature verify
  src/kernel_core/process_spawn/capsule_spawn/runner/verified.rs   spawn_verified
  src/kernel_core/process_spawn/capsule_spawn/runner/preflight.rs  the two stages
```

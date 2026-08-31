# The self-verifying build

`make` on NONOS does not end when the image is packaged. It ends when the build
has proven, against the artifacts it just wrote, that what is about to ship is
what the boot chain will enforce. The same checks run standalone as
`make verify`, so anyone holding the tree can re-establish the result without
trusting the machine that built it.

This page describes what is checked, what each check actually proves, and what
none of them prove. The recipe is `nonos-mk-verify-image` in `mk/20-build.mk`;
the STARK verifier lives in `nonos-stark-enroll/src/main.rs`; the receipt tool
is `scripts/build_receipt.py`.

## Why a build verifies itself

An attested operating system that is built by an unattested build has a gap in
the middle of its story. The kernel proves every capsule at spawn, but between
`cargo` and the ISO there are signatures, trailers, capability declarations,
and enrollment roots produced by half a dozen tools, and any of them can drift:
a capsule rebuilt after enrollment no longer matches its proof, a manifest
re-signed with the wrong caps grants what the source never declared. Those
drifts are exactly what the five checks catch, and they are caught at build
time, on the builder's machine, not at boot on a user's.

These are not hypothetical failure classes. Each check exists because the
class it catches occurs in practice in any tree that rebuilds artifacts after
proving over them, and a verifier that has never refused anything should be
assumed untested rather than lucky.

## The five checks

Each check has a different source of truth, so no single compromised input
passes quietly. They run in order and fail closed: the first failure stops the
build from calling itself ready.

**A. The trust artifact ledger.** Every file under `nonos-data/trust`
(capsule certificates, manifests, trailers, publisher keys, policy roots) is
hashed and compared against `MANIFEST.sha256`. The build re-stamps the ledger
after enrollment (`nonos-mk-trust-ledger`, deterministic order), and a
published tree carries the stamped ledger, so from publication onward this
check pins the shipped trust set byte for byte. What it proves: the trust
directory is exactly the one the ledger describes. What it does not prove:
that the ledger itself is honest; that comes from it being committed and
reviewed like any other source.

**B. Manifest signatures.** Every capsule manifest is verified with
`capsule-sign verify-manifest` against its identity certificate and the baked
trust anchor policy: an Ed25519 and an ML-DSA-65 signature both have to hold.
What it proves: each manifest was signed by the publisher key the trust anchor
recognises, and has not been altered since. The ML-DSA-65 half keeps this true
against a future quantum adversary.

**C. Declared capabilities.** Every built ELF's `.nonos.caps` section is
compared against the capability mask its manifest grants
(`scripts/check_declared_caps.py`). What it proves: no binary ships asking for
less than its manifest grants while the manifest smuggles more; the authority
the kernel will install is the authority the source declared. Checks B and C
together close the loop manifest-to-binary from two independent directions.

**D. STARK membership proofs.** Every capsule's attestation trailer is
re-verified against the policy root, and the kernel's own trailer against the
kernel-attest root, using `nonos-stark-enroll verify` and `verify-kernel`.
This is not a parallel implementation: the subcommands call the same
`gate_verify` parse and the same `nonos-stark` verifier that the kernel runs
at spawn and the bootloader runs before the jump. One verifier, three call
sites. The proof context binds the capsule's BLAKE3 measurement, its granted
capability mask, and the policy epoch, so a swapped binary, an altered cap
mask, or a replayed old epoch each fail the same check. What it proves: the
artifacts on disk are the exact set the transparent post-quantum proofs were
issued for. The soundness of the proof system itself is inherited from its
components; see the proof-system pages for what is and is not machine-checked.

**E. Root embedding.** The policy root that check D verified against is
searched for, byte for byte, inside the shipped kernel image, and the
kernel-attest root likewise (`scripts/build_receipt.py`). Without this, checks
A through D only establish internal consistency of files in a directory; with
it, the proofs are anchored to the artifact that actually boots, because the
kernel enforces the root that is compiled into it and no other. A root that
verifies but is absent from the kernel is a hard failure.

## The receipt

When A through D have passed, the receipt tool writes
`target/attestation/build-receipt.json`: commit, branch, tree-dirty flag,
`SOURCE_DATE_EPOCH`, toolchain, host, the SHA-256 (and BLAKE3 where available)
of the kernel image, the ISO and the ESP kernel, both roots, and the
root-embedding verdict. Two properties are deliberate:

- Everything in it is measured from the tree at receipt time, never echoed
  from build variables, so a stale artifact disagrees with its receipt instead
  of hiding behind it.
- The verification field records only what the script itself measured (root
  embedding); the four earlier gates are listed as preconditions, because a
  receipt existing at all means make reached E. Nothing in the receipt claims
  more than its author checked.

The receipt is a measured record, not yet a signed one. Signing it under a
release key, and reproducing the artifacts it names from a pinned environment,
are the two steps between this and a claim a third party can verify with no
trust in the builder at all. The `flake.nix` at the repository root is the
start of the second: `nix develop` pins the whole tool environment by lock
file, and turning the capsule set into content-addressed derivations is the
stated next step.

## Engineering notes that are easy to get wrong

- **Verification never mutates.** The ledger re-stamp runs in the build chain
  before verification, never inside `make verify`. A verifier that modifies
  its inputs is not a verifier.
- **No pipes around verdicts.** Piping a verifier's output through anything
  replaces its exit status with the last command's and turns a failing proof
  into a passing build. The recipes run the verifiers bare; their per-artifact
  output is the evidence, and their exit codes are the gate.
- **Randomized proofs, stable claims.** STARK trailers embed grinding nonces,
  so re-enrollment changes every trailer byte-for-byte while proving the same
  statement. Nothing may compare trailers by hash across enrollments; the
  ledger pins a published set, and verification re-runs the math.

## The cost model, measured

Numbers from a 4-core reference host; better hosts scale with cores because
enrollment proves in parallel.

| Operation | Cost | When it is paid |
|---|---|---|
| Verify the whole image (A..E, 87 capsules) | ~15 s | end of every `make`; any `make verify` |
| One STARK membership verify | milliseconds | per capsule inside D |
| Enroll (prove) the 87-capsule set | ~10 min | once per release-shaped build |
| Kernel self-attestation proof | minutes | same |

Verification is cheap and proving is expensive, which is the correct asymmetry
for a transparent proof system, and it dictates the workflow: proving is a
release cost, never an iteration cost. One consequence is structural: the
policy tree commits to the entire capsule set, so changing a single capsule
changes the root and invalidates every trailer. That is by design; the root is
one 32-byte statement about the whole system, and there is no sound way to
keep 86 old proofs alive under a new root.

So the build has two loops, and they are honest with each other:

```
make dev        # the iteration loop: rebuild + sign what changed, boot with
                # stale proofs tolerated (rollout mode), labelled at build
                # and at every spawn. Seconds to low minutes.
make dev-qemu   # boot the dev image just built

make            # the proving loop: enroll, build, sign, attest, package,
                # then verify A..E and write the receipt. This is the only
                # path allowed to call the image ready.
make verify     # re-prove an already-built image; it will correctly refuse
                # a dev tree, which is the system working
```

Nothing in the dev loop weakens the release path: signatures and capability
checks still run in dev, only the membership proofs may be stale, and the boot
log names every capsule that rode a stale proof. `make verify` on a dev tree
fails by construction, so a dev image can never masquerade as a proven one.

## Running it

```
make            # build, then prove: A B C D E, then the receipt
make verify     # the same five checks against the artifacts already in the tree
make dev        # the fast loop, proofs stale by design, loudly labelled
```

Timings are printed per check by the verifier itself, so the cost claims in
this page are re-measured on every run rather than trusted to it.

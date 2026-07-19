# Verification

This page states exactly what NONOS proves, what it does not, and how to reproduce every claim from a
clean checkout. It is written to be audited, not believed. The machinery it summarizes lives under
`verification/` and in the proof crates under `userland/*_proofs/`, and the deeper narrative is in
`verification/ARCHITECTURE.md`; this page is the wiki-level map of it.

## The thesis: proofs over the code that runs

A proof is only as strong as the distance between the thing proved and the thing that boots. A kernel
that carries a formal model and proves theorems about that model has established something real, but
whether the code that actually runs refines the model is a separate question. For a total-correctness
effort like seL4 that question is itself answered by a machine-checked refinement proof from the model
down to the C. For most projects that claim "formal verification" it is not answered at all, and that
unproven gap is where defects live.

NONOS closes the gap a different way for the properties it proves: the runnable proofs include the
**real `src/` and capsule source, unmodified, through Rust's `#[path]` mechanism** (only the syscall
clock is shimmed), and execute it. Where a property is naturally an abstract theorem it is stated in
Lean, and a second proof, in Verus over the real bit-operations or in a runnable proof over the real
code, shows the implementation satisfies it. The model is never left standing on its own. NONOS does
not claim total functional correctness of the whole kernel; it claims the security-critical properties,
proven over the running code, with nothing left as an unproven placeholder.

## The layers, strongest at the bottom

**Layer 0, source hygiene.** `nonos-verify hygiene` scans all production Rust under `src/` and
`userland/` and fails the build on panic paths (`unwrap`, `expect`, `panic!`), stub macros (`todo!`,
`unimplemented!`, `unreachable!`), dead-code allow markers, and temporary comment markers. Proof crates
are excluded because their assertions are allowed to panic. This is not a proof of correctness; it is a
machine-enforced floor that the production kernel contains no panic path and no stubbed logic, re-checked
on every push.

**Layer 1, runnable proofs over the real source.** Host crates that `#[path]`-include the actual kernel
and capsule code and run it with `cargo test`:

- `userland/fs_proofs` (58 passing): VFS store operations, path-security canonicalization and the
  `/capsules` read-only guard including slash-smuggling, the protocol codec against hostile input,
  caller attestation rejecting userspace impersonation, and fuzz proofs asserting the parsers never
  panic and never violate their invariants over millions of structured and random inputs. Writing these
  found and fixed real bugs.
- `userland/crypto_proofs`: the real kernel crypto checked against standard vectors, SHA-256/512
  (FIPS 180-4), SHA-3 (FIPS 202), BLAKE3, HMAC-SHA-256 (RFC 4231), HKDF (RFC 5869), ChaCha20-Poly1305
  (RFC 8439), AES-GCM (NIST), Ed25519 (RFC 8032), P-256/P-384 ECDSA, secp256k1, and RSA, each including
  tamper rejection.
- `userland/net_proofs`, `driver_proofs`, `stark_proofs`, `kernel_proofs`, `usb_proofs`, and the
  in-image proof capsules (`capsule_gui_proof`, `capsule_std_proof`, `capsule_input_proof`,
  `capsule_proof_io`), each asserting a specific guarantee over real code.

**Layer 1b, bounded model-checking (Kani).** Several proof crates carry Kani harnesses
(`nonos-bootloader/boot_proofs/src/kani_proofs.rs`, `userland/fs_proofs/src/kani_proofs.rs`,
`userland/driver_proofs`, and `userland/stark_proofs/src/kani_proofs.rs`, which proves the untrusted
attestation-trailer deserializer is total on any input). Kani exhaustively checks a function over all
inputs within a bound, which is
stronger than testing for the bounded region and catches the arithmetic and boundary cases fuzzing can
miss. It is bounded, not unbounded: it proves the property for inputs up to the bound, not for all
inputs of unbounded size.

**Layer 2, Lean theorems.** The Lean files under `verification/lean/Nonos/` carry **887 theorems across
120 modules with zero `sorry`** (Lean's placeholder for an unproven step), so every stated theorem is
fully proven. The tree is core-only: no Mathlib, so any recent Lean 4 toolchain checks it. The live
counts are in [the evidence manifest](#the-evidence-manifest), regenerated and checked on every push.

The modules map one to one onto the trusted path. A representative slice:

```
  Capability / CapMask / CapToken   grant/revoke/attenuate algebra and delegation-subset: a delegated
                                    mask never carries a capability its parent lacks; a token is
                                    admitted only if signed, unexpired and unrevoked
  UserCopy                          the user/kernel boundary: an accepted copy range lies wholly in
                                    user space, never the null page or the kernel half
  Paging / Isolation / DemandPaging no writable-and-executable page; a served demand page is never
                                    executable; no kernel-half address is demand-backed
  AntiRollback / AntiRollbackState  a monotone version floor that never falls; the concrete bootloader
                                    check refines the abstract theorem
  DmaMap / IrqBind / MsixExclusion  the hardware broker: bounded DMA owned on a fresh epoch, a bounded
                                    MSI-X bind, and no raw mapping ever reaching a protected register
  ElfPhdr / ElfReloc / LoadProtect  the loader: in-bounds program headers, a bounded relocation write,
                                    RELRO sealed read-only before entry
  FdAlloc / PidAlloc / SyscallRoute the allocators and the first-match syscall dispatch
  Wpa2Handshake / CcmpReplay        the Wi-Fi trusted path: keys install only on a valid message 3, a
                                    replayed packet number is dead forever
```

The 168 flagship theorems are listed in `verification/lean/AxiomProfile.lean`, which prints each one's
exact axiom closure. The CI Lean job runs it as a gate: a closure may name only Lean's three standard
axioms (`propext`, `Classical.choice`, `Quot.sound`) and never `sorryAx`, so a `sorry` anywhere in a
proven theorem's dependency graph fails the build. This is machine-enforced, not a convention.

The transparent STARK attestation adds a second body of proof, 203 theorems across the
`Nonos/Stark/` modules plus `SigningKey` and `KeyLifecycle`: Merkle membership soundness and
collision-freedom, the binding of a proof to its capsule identity, capabilities, policy epoch and
domain, length-prefixed measurement injectivity, the money-grade soundness budget, trailer parse
safety, and signing-key rollback, revocation and validity windows. These pin the model the STARK
spawn gate refines; see [attestation](../security/attestation.md) and the
[proof system](../subsystems/proof-system/stark.md).

**Layer 2b, Verus refinement.** `verification/verus/` proves that the real Rust bit-operations match
the Lean model: the capability `has`/`grant`/`revoke`/`attenuate` functions (`revoke_is_monotonic`,
`revoke_drops_the_right`, and their companions), the page-permission spec, and the IPC-length spec are
proven in Verus directly over the Rust semantics. The attestation adds `stark_attestation.rs`, which
SMT-checks that a trailer length capped at the bytes remaining never over-reserves and that the gate
accepts only the conjunction of the root, context and enrollment checks. This is the bridge that ties
the abstract Lean theorem to the concrete `bits & !bit` the kernel executes.

**Layer 2c, mechanical extraction (Charon and Aeneas).** This is the strongest link between proof and
code, and it removes the transcription step entirely. [Charon](https://github.com/AeneasVerif/charon)
lowers real kernel functions from the MIR that `rustc` compiles into LLBC, and
[Aeneas](https://github.com/AeneasVerif/aeneas) translates that into a pure Lean definition. The Lean
theorems in `verification/extraction/lean/` are then proven directly on that extracted definition, so
the thing proved is the real function, not a model a human retyped from it. The extraction crates under
`verification/extraction/` include the real `src/` files unmodified through `#[path]`; nothing is
copied. Extracted and proven so far:

```
  capabilities::bits::{has,add,remove}_capability   the capability word operations; extracted_remove_confines
                                                    proves the extracted revoke never grants a new capability
  usercopy::policy::check_range                     the user/kernel range check; check_range_spec proves it
                                                    accepts exactly the non-null, in-bound, non-wrapping ranges,
                                                    tied to the abstract Isolation.Accepts
  memory::paging is_wx_violation, to_pte_flags      the W^X page encoder; extracted_no_wx_page proves the
                                                    extracted encoder never emits a writable-executable page
  broker::irq::validate_msix_request                the MSI-X bind validator; a request with an unknown flag
                                                    bit is refused before any device state is read
```

The extraction is regenerated in CI and diffed against the checked-in Lean, so it can never silently
drift from the source. Where Aeneas models a `core` library method as an opaque axiom (for example
`Option::ok_or`), that axiom appears in the theorem's closure and is documented as such; it is a
faithful stand-in for the standard method, not a `sorry`.

## The evidence manifest

`verification/EVIDENCE.json` is a machine-readable inventory of the whole verification surface,
generated from the source tree by `verification/collect-evidence.sh` and checked for drift on every
push. It carries the live counts (Lean modules and theorems, the `sorry` count, the axiom-profiled
theorems, the extracted functions with their file hashes, and the Verus, Kani and runnable-proof
totals) and the pinned toolchains. As of this writing:

```
  Lean specification    120 modules, 887 theorems, 0 sorry, 168 axiom-profiled
  mechanical extraction  7 functions lowered from real MIR (Charon + Aeneas)
  Verus refinement       5 source files over the real Rust bit-operations
  Kani model checking    65 harnesses
  runnable proofs        32 proof crates over the real source
```

Because the manifest is regenerated and diffed in CI, these numbers in the repo are always current and
CI-checked, not a claim in prose that can rot. The green Lean, Verus, Kani and extraction jobs are the
proof that the surface actually checks; the manifest is the inventory of it.

## What is established

Over the real code, machine-checked, reproducible, and re-run on every push:

- The **capability algebra** is sound: authority only shrinks under attenuation and revoke, grant never
  removes, and the operations compose as the model says (Lean, plus Verus over the real bit-ops).
- **Address-space isolation** and **page-permission** invariants hold (Lean, Verus): no
  writable-and-executable page, no capsule reach outside its mappings.
- **Freed memory is zeroized** (Lean, plus the runnable zeroization checks): no cross-tenant residue.
- **Attestation and anti-rollback** reject the substitution and rollback cases (Lean, plus the runnable
  attestation proofs rejecting impersonation).
- **Path canonicalization** and the `/capsules` guard resist smuggling (Lean, plus fs_proofs fuzz).
- The **parsers never panic and never violate their invariants** over millions of hostile inputs (fuzz).
- The **crypto primitives** conform to their standard vectors and reject tampering (crypto_proofs).
- The **production source carries no panic path or stub** (hygiene).

## What is NOT established

Stated plainly, because an honest scope is the point:

- **Not total functional correctness.** NONOS does not prove that the entire kernel does exactly what a
  full specification says, the way seL4 does. It proves the security-critical properties above, not
  every behavior of every subsystem. A subsystem can be correct against these properties and still have
  a functional bug outside them.
- **Kani is bounded.** The model-checked properties hold for inputs up to the harness bound, not for all
  unbounded inputs.
- **Known-answer vectors are conformance, not a universal proof.** Passing FIPS/RFC/NIST vectors shows
  the implementation agrees with the standard on those vectors and rejects tampering; it is strong
  evidence of correctness but is not a proof of the algorithm for every possible input.
- **Side channels are out of scope of these proofs.** The crypto is portable software; the [crypto
  pages](../subsystems/crypto/README.md) state honestly where a primitive is not constant-time. The
  proofs are functional and structural, not timing proofs.
- **The hardware is trusted below the IOMMU line.** With the IOMMU backend not engaged, the DMA safety
  argument rests on the broker's software bounds plus non-malicious device hardware; this is stated on
  the [DMA](../subsystems/hardware-broker/dma.md) page.

## Versus seL4, and versus marketing

**Versus seL4.** seL4 is the gold standard for total kernel verification: it proves full functional
correctness of the whole kernel in Isabelle/HOL and proves that the C refines that model. NONOS does
not match that scope and does not claim to. What NONOS does differently is (1) prove a focused set of
security-critical properties rather than total correctness, (2) run those proofs over the actual Rust
source rather than only an abstract model, and (3) build on a memory-safe language, which removes by
construction a large class of the memory-corruption bugs a C kernel's proof must rule out. The two are
different points on the same spectrum: seL4 proves everything about a minimal C kernel; NONOS proves the
things that matter most about a larger Rust system, over the code that runs, and is honest that the rest
is tested rather than proven.

**Versus "formally verified" marketing.** The common failure is to prove a model with no link to the
running code, or to leave theorems as `sorry`, or to call a test suite a proof. NONOS's Lean carries
zero `sorry`, its Verus proofs are over the real bit-operations, its runnable proofs include the real
source, and every claim on this page is reproducible from a clean checkout by the commands below. The
scope is narrower than the marketing usually implies and stated as such.

## Reproduce it

```sh
# Layer 0: source hygiene
cargo run --manifest-path nonos-verify/Cargo.toml -- hygiene

# Layer 1: runnable proofs over the real source
cd userland/fs_proofs      && cargo test --release
cd userland/crypto_proofs  && cargo test --release

# Layer 1b: bounded model-checking
cd userland/fs_proofs      && cargo kani --output-format terse

# Layer 2: Lean theorems (requires the Lean toolchain in verification/lean)
cd verification/lean       && lake build
# the axiom gate: prints each flagship theorem's axiom closure, must show no sorryAx
cd verification/lean       && lake env lean AxiomProfile.lean
# the reproducible proof-corpus commitment (refuses to form on any sorry)
./verification/lean/proof-corpus-root.sh

# Layer 2c: mechanical extraction (requires Charon, Aeneas, and the Lean toolchain)
cd verification/extraction/lean && lake exe cache get && lake build

# The evidence manifest: regenerate and confirm it matches the committed copy
./verification/collect-evidence.sh | diff - verification/EVIDENCE.json
```

## Source map

```
  verification/EVIDENCE.json        the machine-readable inventory, CI-checked for drift
  verification/collect-evidence.sh  regenerates the manifest from the source tree
  verification/README.md            the layered framing and the run commands
  verification/ARCHITECTURE.md      the thesis, threat model, and what is / is not established
  verification/lean/Nonos/*.lean    the 887 Lean theorems across 120 modules (zero sorry)
  verification/lean/AxiomProfile.lean   the 168-theorem axiom gate (no sorryAx, standard axioms only)
  verification/lean/proof-corpus-root.sh  the reproducible commitment over the proven corpus
  verification/extraction/           Charon + Aeneas crates and the proofs on the extracted MIR
  verification/verus/src/*.rs        the Verus refinement of capabilities, paging, IPC lengths
  userland/fs_proofs/                runnable + Kani proofs over the real VFS/parser/attestation code
  userland/crypto_proofs/            the crypto known-answer and tamper-rejection proofs
  userland/*_proofs/                 32 proof crates, one guarantee per subsystem, over the real source
  nonos-verify/                      the source-hygiene gate
```

Every reference above is verified against those trees. The properties proven here are what the
[mission](mission.md) rests on, the capability algebra is specified on the [capabilities
page](../security/capabilities-and-tokens.md), the attestation on the [attestation
page](../security/attestation.md), and the crypto primitives on the [crypto
pages](../subsystems/crypto/README.md).

## Get involved

The verification surface grows one machine-checked property at a time, and the on-ramp is real. Good
first contributions, roughly easiest first:

- **Add a Lean theorem over a real invariant.** Pick a small, self-contained function in `src/` (a
  bounds check, an allocator, a validator), model it faithfully in a new `verification/lean/Nonos/`
  module, and prove its safety property. The tree is core-only, so no Mathlib to learn; the existing
  modules are the template. Add the flagship theorems to `AxiomProfile.lean` and confirm the axiom
  closure stays clean.
- **Strengthen a runnable proof.** The `userland/*_proofs/` crates run the real source. A new assertion,
  a tighter invariant, or a fuzz harness that finds a real bug is as welcome as a new theorem.
- **Mechanically extract a function.** Add a real kernel function to an extraction crate under
  `verification/extraction/`, regenerate with Charon and Aeneas, and prove its property on the extracted
  definition. This is the strongest kind of contribution because it closes the model-to-code gap.
- **Extend the evidence and CI.** New proof systems, new drift checks, better badges: the manifest and
  the workflows are the project's public statement of what it proves.

The [contributing guide](../community/contributing.md) covers the house rules; the reward program in
[rewards](../community/rewards.md) weights work on the trusted path and on verification highest.

## FAQ

**Is this "formally verified" like seL4?** No, and the page says so plainly above. seL4 proves total
functional correctness of a minimal C kernel. NONOS proves a focused set of security-critical
properties, over the real Rust source, on a memory-safe language. Different scope, honestly stated.

**How many theorems are there, really?** 887 Lean theorems across 120 modules with zero `sorry`, plus
203 in the STARK attestation body, plus the Verus, Kani and runnable proofs. The live counts are in
`verification/EVIDENCE.json`, regenerated and diff-checked on every push, so the number in the repo is
never stale.

**What does "zero sorry" actually guarantee?** A `sorry` is Lean's placeholder for an unproven step. If
any proven theorem depended on one, its axiom closure would name `sorryAx`, and the CI axiom gate
(`AxiomProfile.lean`) fails on exactly that. So every stated theorem is fully proven under at most
Lean's three standard axioms.

**How is the Lean tied to the code that runs?** Three ways, strongest last: the runnable proofs execute
the real `src/` through `#[path]`; the Verus proofs run over the real Rust bit-operations; and the
mechanical extraction lowers real functions from `rustc`'s MIR and proves on that. Where a property is
only an abstract Lean theorem, a second proof connects it to the implementation.

**Can I trust the numbers on this page?** They are checked. `verification/EVIDENCE.json` is regenerated
from the source and diffed in CI, so a stale number fails the build. Every command in "Reproduce it"
runs from a clean checkout.

**What is NOT proven?** Total functional correctness, unbounded model checking, timing side channels,
and hardware below the IOMMU line. The "What is NOT established" section above is the honest scope.

**Where do I start reading the proofs?** `verification/lean/Nonos/Capability.lean` for the algebra,
`UserCopy.lean` for a boundary check, and `verification/extraction/lean/NonosExtraction/Refinement.lean`
for a proof landed on mechanically-extracted code.

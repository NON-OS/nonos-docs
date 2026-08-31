# The STARK

The kernel carries a transparent STARK: a proof system that lets a prover convince a verifier that a
computation satisfies a set of algebraic constraints, with no trusted setup, resting only on the field
and a hash. This page documents the prover, the verifier, and the low-degree test they share. The code
lives in the `nonos-stark` crate, shared with the bootloader and re-exported into the kernel as
`crate::crypto::stark` (`src/crypto/mod.rs:36`, `pub use nonos_stark as stark`); the files are under
`nonos-stark/src/air/` and `nonos-stark/src/fri/`.

## The AIR

A computation is stated as an Algebraic Intermediate Representation, the `Air` trait
(`nonos-stark/src/air/spec.rs:28`):

```
  trait Air:
      log_trace_len()      -> u32     // log2 of the number of steps
      trace_width()        -> usize   // columns (state registers)
      transition(frame)    -> [Fp]    // constraints that must vanish between consecutive rows
      boundary()           -> [...]   // public conditions on the first / last rows
      constraint_degree()  -> usize   // the maximum constraint polynomial degree
```

The trait also carries `window_size()` (how many consecutive rows a transition constraint reads) and
`num_transition()` (`nonos-stark/src/air/spec.rs:37`, `:45`), and a companion `AirExt` trait
(`spec.rs:75`) adds `transition_ext` over the quadratic extension `Fp2`, which the attestation AIRs use.
The trace is a table: `trace_width` columns and `2^log_trace_len` rows, one row per step of the
computation. The transition constraints are the rules that must hold between each row and the next
(for example "the next value is this value cubed"), and the boundary constraints pin the public
inputs and outputs on the first and last rows. A specific computation is an `Air` impl; the
[catalog](air-catalog.md) is the set of them.

## The prover

`stark_prove` (`nonos-stark/src/air/prove.rs:54`) turns a satisfying trace into a proof:

```
  stark_prove(air, trace):
      1. low-degree extend each trace column over a coset (shift 7) of a larger domain
      2. Merkle-commit each extended column; absorb the roots into the transcript
      3. draw random composition coefficients from the transcript
      4. build the composition polynomial: fold the transition and boundary
         quotients under those coefficients into one polynomial
      5. draw an out-of-domain point z from the transcript (off both domains)
      6. open the trace frame at z; build the DEEP quotient binding the
         committed columns to that out-of-domain frame
      7. run FRI on the DEEP quotient to prove it is low degree
```

Each step is standard and each is what makes the proof sound. The trace is extended to a larger domain
so a low-degree test is meaningful; the Merkle commitment binds the prover to the columns before it
sees any challenge; the random composition coefficients collapse many constraints into one polynomial
that is low-degree only if every constraint held; the out-of-domain point `z`, drawn after the
commitments, forces the prover to answer at a point it could not have prepared for (the DEEP-ALI
technique); and FRI proves the resulting quotient really is low degree.

## The verifier

`stark_verify` (`nonos-stark/src/air/verify.rs:40`) mirrors the prover exactly:

```
  stark_verify(air, proof):
      1. absorb the claimed roots; redraw the same coefficients, z, and DEEP coefficients
         (identical Fiat-Shamir, so prover and verifier agree by construction)
      2. recompute the composition value at z from the claimed frame, using the AIR's own algebra
      3. FRI-verify the DEEP quotient is low degree
      4. for each sampled query: check the Merkle openings against the committed roots,
         reconstruct the DEEP value from them, and match it to the FRI query
```

The verifier never trusts the prover's claimed evaluations; it recomputes the constraint algebra itself
at the out-of-domain point and checks the openings against the committed roots. The Fiat-Shamir
transcript is what makes the whole thing non-interactive: both sides derive every challenge by hashing
the transcript so far, so a dishonest prover cannot choose a challenge to its advantage.

## FRI

FRI is the low-degree test at the core, and the verifier's soundness rests on it
(`nonos-stark/src/fri/verify.rs`). The prover commits a codeword, and repeatedly folds it in half under
a transcript challenge, committing each layer, until a constant remains; the verifier redraws the fold
challenges, **checks the final layer is a single constant** (`fri/verify.rs:63`), and for each query
re-derives the fold from the openings and checks it matches the next layer's committed value
(`fri/verify.rs:96`):

```
  fri_verify(proof):
      redraw fold challenges from the transcript
      require final_layer is constant           // a high-degree codeword folds to a non-constant w.h.p.
      for each query: recompute the fold from the openings; match the next layer
```

If the original codeword were not low degree, folding it would not collapse to a constant except with
negligible probability, so the constant-final-layer check plus the per-query fold consistency is the
low-degree guarantee.

## Two FRI variants

There are two FRIs (`nonos-stark/src/fri/` and `fri_poseidon/`), differing only in their hash: the
default uses BLAKE3 for the Merkle commitments and the Fiat-Shamir transcript, while `fri_poseidon`
uses the [Poseidon](field-and-poseidon.md) permutation for both. The Poseidon variant is slower per hash
but algebraic, which means the verifier itself can be expressed as STARK constraints and proven inside
another proof. That is the door to recursion, covered on the [AIR catalog](air-catalog.md) page.

## The attestation parameters

When this STARK is used for [attestation](../../security/attestation.md), its soundness is set by
explicit constants in one place, `nonos-stark/src/attest_params.rs`, so that a prover and a verifier that
link the same crate cannot drift and a downward change in any of them is a visible edit:

```
  N_QUERIES         = 32     (attest_params.rs:41)   FRI query count
  GRIND_BITS        = 16     (attest_params.rs:45)   transcript proof-of-work bits
  EXTRA_BLOWUP_BITS = 3      (attest_params.rs:48)   blowup beyond the AIR's own rate
  LOG_ROUNDS        = 5      (attest_params.rs:37)   the attestation hash runs 2^5 = 32 full rounds
```

These are the real FRI soundness knobs, not a vibe: the query count and the grinding bits set the
per-query and the grinding cost a forger must beat, and the extension-field challenges (drawn over `Fp2`)
add to the soundness error's field size. They can be dialled up. The [attestation page](../../security/attestation.md)
covers where the constants are read; this page is where the construction they parameterise is documented.

## Honest scope

What this system is: a complete, transparent STARK, prover and verifier, with the standard sound
components (low-degree extension, Merkle commitment, random-coefficient constraint composition,
DEEP out-of-domain binding, and a FRI low-degree test), needing no trusted setup and no parameters
beyond the field and the hash. What it is not: it does not ship a machine-checked proof of its own
soundness, and the kernel is `no_std` so there are no in-tree unit tests of the STARK itself; its
security is the security of these well-studied components as implemented here. The negative testing
that does exist is on the [attestation verifiers](pedersen-attestation.md), which are fuzzed to reject
random proofs over thousands of adversarial inputs.

## Security analysis

The STARK's job is to be a verifier a dishonest prover cannot fool, and the way it fails is as important
as the way it passes.

**The verifier recomputes, it never trusts.** `stark_verify` (`air/verify.rs:40`) redraws every
Fiat-Shamir challenge from the transcript itself, recomputes the composition value at the out-of-domain
point from the AIR's own algebra, and checks each query's Merkle openings against the committed roots. It
does not read a claimed pass/fail from the proof. This is what makes soundness structural rather than
polite: a proof is accepted only because the verifier reran the constraint algebra and it matched, so a
forged evaluation has to survive the recomputation, not just be asserted.

**Every check is fail-closed, and the return is a bare `bool`.** `stark_verify` returns `false` the
moment anything is off: the wrong shape (`air/verify.rs:61`, the root, frame, and query counts must match
the AIR), a Merkle opening that does not reconstruct, or a FRI query that does not match. FRI itself
requires the final layer to be a single constant (`fri/verify.rs:63`) and rejects otherwise, which is the
low-degree conclusion, because a high-degree codeword folds to a non-constant with overwhelming
probability. There is no partial-credit path and no exception that leaves the proof half-accepted; the
function is a total predicate that says yes only when everything held.

**The honest boundary is that soundness is inherited, not proven in-tree.** As the scope section states,
this system ships no machine-checked proof of its own soundness and, being `no_std`, carries no in-tree
unit tests of the STARK itself. Its security is the security of low-degree extension, Merkle commitment,
random-coefficient composition, DEEP out-of-domain binding, and the FRI test as implemented here. The
negative testing that does exist is on the [attestation verifiers](pedersen-attestation.md), fuzzed to
reject random proofs over thousands of adversarial inputs. So the claim is "a faithful implementation of
sound components," not "a formally verified verifier," and this page says so rather than implying more.

## Debugging the STARK

Because `stark_verify` returns only `true` or `false`, a rejection carries no message of its own; you
localize it by the shape check it fails and by the runtime path that consumes the result.

**Where a rejection actually surfaces.** In the running kernel the STARK and zk verifiers are reached
through capsule attestation, and that path prints. The [attestation gate](../../security/attestation.md)
emits `[ZK-ATTEST] ok`, `[ZK-ATTEST] FAIL`, or `[ZK-ATTEST] none` on the serial console with the capsule
name (`src/kernel_core/process_spawn/capsule_spawn/runner/attest_gate.rs`), so a proof that fails to
verify shows up there, not inside the prover. That marker is the first thing to read when a signed capsule
will not spawn.

**Splitting a bare `false` by cause.** The early shape check (`air/verify.rs:52`) rejects a proof whose
root count, out-of-domain frame length, or query count does not match what the AIR declares, which is the
signature of a proof built against a different AIR shape than the one verifying it, a serialization or
version mismatch rather than a soundness failure. A proof that passes the shape check but still returns
`false` failed either a Merkle opening or the FRI consistency, meaning the committed data and the claimed
evaluations do not agree, which is the genuine "this is not a valid proof of this statement" case. The
constant-final-layer check in FRI (`fri/verify.rs:63`) is the specific spot a not-actually-low-degree
codeword dies, so a prover bug that produces a too-high-degree quotient surfaces exactly there.

## Source map

```
  nonos-stark/src/air/spec.rs         the Air trait
  nonos-stark/src/air/prove.rs         stark_prove
  nonos-stark/src/air/verify.rs        stark_verify
  nonos-stark/src/air/composition.rs   the constraint composition
  nonos-stark/src/fri/                  the BLAKE3 FRI (prove, fold, verify)
  nonos-stark/src/fri_poseidon/         the recursion-friendly Poseidon FRI
  nonos-stark/src/transcript.rs, poseidon_transcript.rs   the Fiat-Shamir transcripts
  src/kernel_core/process_spawn/capsule_spawn/runner/attest_gate.rs   the [ZK-ATTEST] marker a rejection surfaces through
```

Every reference above is verified against those trees. The AIRs this prover and verifier operate on are
cataloged on the [AIR catalog](air-catalog.md) page, the field and hash they are expressed over are on the
[field and hash](field-and-poseidon.md) page, and the runtime gate that consumes a verification result and
turns a `false` into a refused spawn is on the [capsule attestation](../../security/attestation.md) page.

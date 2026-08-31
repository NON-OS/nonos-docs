# The AIR Catalog and Recursion

The [STARK](stark.md) proves that a trace satisfies an `Air`. The catalog is the set of `Air`
implementations the kernel carries, from small arithmetic gadgets to the composite AIR that is a
verifier in itself. This page documents them and how they compose toward recursion. The code is under
`nonos-stark/src/air/`.

## The gadgets

Each gadget is an `Air` that proves one kind of statement, and their constraint degrees span the range
the prover must handle:

```
  fibonacci        f[i+2] = f[i+1] + f[i]                        degree 1   (a linear recurrence)
  squaring         f[i+1] = f[i]^2                                degree 2
  power_chain      f[i+1] = f[i]^7 + c                            degree 7   (exercises the high-degree path)
  poseidon         one row per Poseidon round; proves a preimage  degree 7
  merkle_membership a Poseidon Merkle path from a leaf to a root   degree 7
  multi_membership  several Merkle openings under one root         degree 8   (a FRI-query verifier)
  fri_fold          a FRI fold step is correct                     degree 4
  trace_fold        a FRI fold with the challenge witnessed in the trace   degree 4
  permutation       multiset equality via a grand product          degree 3
  copy_constraint   a Plonk-style wiring permutation forces equalities    degree 3
  fiat_shamir       a Poseidon transcript challenge was squeezed correctly   degree 7
  range_check       a value lies in a bounded range                degree 2
  lookup            a value is one of a committed table's entries   degree 3
```

The degrees are read from each gadget's `constraint_degree` (for example `fri_fold.rs:95`,
`multi_membership.rs:284`, `copy_constraint.rs:108`, `range_check.rs:59`, `lookup.rs:129`). The catalog
also carries `accumulator` and `poseidon_preimage`, plus quadratic-extension variants of several gadgets
(the `*_ext` files) that the attestation AIRs use.

The small ones (fibonacci, squaring, power_chain) are the arithmetic backbone and exist to exercise the
prover at each constraint degree. The consequential ones are the middle group: `poseidon` proves
knowledge of a hash preimage, `merkle_membership` proves a leaf is in a committed Poseidon Merkle tree,
`fri_fold` proves a FRI fold was computed correctly, and `fiat_shamir` proves a transcript challenge was
derived correctly. Those four are exactly the pieces a STARK verifier performs, which is the point.

## Fusing and wiring

Two gadgets exist to combine the others. `fused` (`air/fused.rs`) stacks several AIRs into one trace
with a per-row selector that activates one gadget's constraints at a time, so several computations share
one proof. `wired` (`air/wired.rs`) goes further: it fuses regions and then binds values across them
with a Plonk-style copy constraint, so a value produced in one region (say, a transcript challenge from
the `fiat_shamir` region) is forced equal to the value consumed in another (the fold challenge in the
`fri_fold` region). `wired` is the mega-AIR: the monolithic backbone that stitches the verifier's pieces
into a single constraint system with the data flowing correctly between them.

## Recursion

Put the pieces together and the STARK verifier becomes a statement the STARK can prove. The
[Poseidon FRI variant](stark.md) makes the verifier's hashing algebraic; `merkle_membership` and
`multi_membership` prove its Merkle openings; `fri_fold` / `trace_fold` prove its folds; `fiat_shamir`
proves its transcript challenges were squeezed correctly rather than trusted as public inputs; and
`wired` fuses these regions with copy constraints so the challenges flow from the transcript into the
folds and the openings exactly as a real verifier would run them. The result is a proof that a proof
was verified, which is recursion: a verifier expressed inside the proof system. This is the machinery
the earlier bring-up assembled (the multi-query fan-out, the cross-region binding, the in-circuit
Fiat-Shamir), and it is why the catalog carries a verifier's worth of gadgets rather than just the
arithmetic examples.

## In-circuit Fiat-Shamir

The `fiat_shamir` AIR (`air/fiat_shamir.rs`) deserves its own note because it is the subtle part. A
non-interactive proof's challenges come from hashing the transcript, and a naive recursive verifier
would take those challenges as public inputs and trust them. That AIR instead proves the challenge was
squeezed from the Poseidon sponge: its trace is the sponge state across the absorbed blocks, its
transitions are the Poseidon rounds and the absorb steps, and its boundary constraints pin the first
input and the final squeezed challenge. So the recursive verifier does not trust the transcript; it
proves it, which closes the gap a public-input challenge would leave open.

## Security analysis

The catalog is not a boundary on its own; its security is what its gadgets let the [STARK](stark.md) prove
soundly, and the sharp edge is the difference between an honest gadget and one that leaves a gap.

**The consequential gadgets are exactly a verifier's operations, which is what makes recursion sound.**
`poseidon` proves a preimage, `merkle_membership` and `multi_membership` prove openings under a committed
root, `fri_fold` and `trace_fold` prove folds, and `fiat_shamir` proves a transcript challenge was
squeezed. Those are the steps a STARK verifier performs, so a proof that all of them hold, wired together,
is a proof that a verification ran. The soundness of the recursion is inherited from the soundness of
these constraints being enforced, the same components the STARK page describes, not from anything the
catalog adds on top.

**The in-circuit Fiat-Shamir closes the gap a naive recursion leaves open.** The subtle failure mode of a
recursive verifier is trusting the transcript challenges as public inputs. `fiat_shamir` (`air/fiat_shamir.rs`)
instead makes the sponge state the trace, the Poseidon rounds and absorbs the transitions, and pins the
first input and the squeezed challenge with boundary constraints, so the challenge is proven rather than
trusted. This is the property that matters most in the catalog: without it the recursion would be verifying
a computation whose challenges an adversary could have chosen, and with it the challenges are forced to be
the honest hash of the transcript.

**The honest boundary is that these are constructions, not certified circuits.** Each gadget is an `Air`
whose `transition` and `boundary` are the claimed constraints; the guarantee is that a trace satisfying
them satisfies the stated relation, and that guarantee is only as good as the constraints being complete
and correct. The catalog ships no machine-checked proof that, say, `merkle_membership`'s constraints admit
exactly the valid openings and no others. As with the STARK itself, the negative testing that exists lives
on the [attestation verifiers](pedersen-attestation.md); the gadgets are built to be right, and their
degrees are stated so the prover handles each, but "built to be right" is the claim, not "formally proven
complete."

## Debugging the AIR catalog

A gadget bug shows up as a proof that will not verify, and because each gadget is a distinct `Air` with a
distinct constraint set, the first move is to identify which gadget's constraints are in play.

**Localize by the gadget and its degree.** The [STARK verifier](stark.md) checks shape before algebra, so a
proof whose trace width or length does not match the `Air` it is verified against fails the early shape
check, which usually means the wrong gadget or a mismatched configuration, not a bad witness. A proof that
passes shape but fails the constraint recomputation has a witness that violates one gadget's transition or
boundary, and the constraint degrees in the table are the cue: a failure in a degree-7 gadget (`poseidon`,
`merkle_membership`, `fiat_shamir`) is in the hashing or membership algebra, while a degree-1 or degree-2
failure is in a fold or a permutation gadget.

**Cross-region wiring is where the fused and wired AIRs fail specifically.** `fused` (`air/fused.rs`)
selects one gadget's constraints per row and `wired` (`air/wired.rs`) then binds values across regions with
copy constraints, so a value produced in the `fiat_shamir` region must equal the one consumed in the
`fri_fold` region. If the individual gadgets each verify in isolation but the composite does not, the fault
is in that cross-region binding: the copy constraint is forcing an equality the trace does not actually
satisfy, which is the signature of the challenge not flowing from the transcript into the fold as a real
verifier would run it. That is the wiring, not the arithmetic, and it is the reason `wired` is the piece
most worth isolating when a recursive proof breaks.

## Source map

```
  nonos-stark/src/air/fibonacci.rs, squaring.rs, power_chain.rs   the arithmetic gadgets
  nonos-stark/src/air/poseidon.rs, merkle_membership.rs, multi_membership.rs   hashing and membership
  nonos-stark/src/air/fri_fold.rs, trace_fold.rs                  the FRI-fold gadgets
  nonos-stark/src/air/permutation.rs, copy_constraint.rs          multiset and wiring constraints
  nonos-stark/src/air/fiat_shamir.rs                              the in-circuit transcript
  nonos-stark/src/air/fused.rs, wired.rs, fusion.rs               fusion, the mega-AIR backbone, and the region-stacking helper
  nonos-stark/src/air/range_check.rs, lookup.rs, accumulator.rs, poseidon_preimage.rs   further gadgets
  nonos-stark/src/attest_params.rs                                the attestation soundness constants
```

`fused` and `wired` do not carry a fixed constraint degree; each computes its degree dynamically from
the gadgets it stacks (`fused.rs:107`, `wired.rs:183`). `fusion.rs` is the shared region-stacking helper
both engines use, and `*_ext` variants (`fused_ext.rs`, `wired_ext.rs`, `wired_multi_ext.rs`) run the
same construction over the quadratic extension.

Every reference above is verified against those trees. The prover and verifier that operate on these AIRs,
and the shape-then-algebra check a bad witness fails, are on the [STARK](stark.md) page; the field and the
Poseidon permutation the degree-7 gadgets are built over are on the [field and hash](field-and-poseidon.md)
page; and the fuzzed negative testing is on the [Pedersen attestation](pedersen-attestation.md) page.

# The AIR Catalog and Recursion

The [STARK](stark.md) proves that a trace satisfies an `Air`. The catalog is the set of `Air`
implementations the kernel carries, from small arithmetic gadgets to the composite AIR that is a
verifier in itself. This page documents them and how they compose toward recursion. The code is under
`src/crypto/stark/air/`.

## The gadgets

Each gadget is an `Air` that proves one kind of statement, and their constraint degrees span the range
the prover must handle:

```
  fibonacci        f[i+2] = f[i+1] + f[i]                        degree 1   (a linear recurrence)
  squaring         f[i+1] = f[i]^2                                degree 2
  power_chain      f[i+1] = f[i]^7 + c                            degree 7   (exercises the high-degree path)
  poseidon         one row per Poseidon round; proves a preimage  degree 7
  merkle_membership a Poseidon Merkle path from a leaf to a root   degree 7
  multi_membership  several Merkle openings under one root         degree 7   (a FRI-query verifier)
  fri_fold          a FRI fold step is correct                     degree 1
  trace_fold        a FRI fold with the challenge witnessed in the trace   degree 1
  permutation       multiset equality via a grand product          degree 2
  copy_constraint   a Plonk-style wiring permutation forces equalities    degree 2
  fiat_shamir       a Poseidon transcript challenge was squeezed correctly   degree 7
```

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

## Source

```
  src/crypto/stark/air/fibonacci.rs, squaring.rs, power_chain.rs   the arithmetic gadgets
  src/crypto/stark/air/poseidon.rs, merkle_membership.rs, multi_membership.rs   hashing and membership
  src/crypto/stark/air/fri_fold.rs, trace_fold.rs                  the FRI-fold gadgets
  src/crypto/stark/air/permutation.rs, copy_constraint.rs          multiset and wiring constraints
  src/crypto/stark/air/fiat_shamir.rs                              the in-circuit transcript
  src/crypto/stark/air/fused.rs, wired.rs                          fusion and the mega-AIR backbone
```

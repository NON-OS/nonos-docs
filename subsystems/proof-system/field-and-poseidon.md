# The Field and the Hash

The proof system is built over one field and one arithmetization-friendly hash. Everything else,
the STARK, the FRI test, the Merkle commitments, the in-circuit transcript, is expressed in terms of
these two. This page documents them, and it is precise about what the hash is and is not. The code is
under `src/crypto/stark/field/` and `src/crypto/stark/air/poseidon.rs`.

## The Goldilocks field

The field is Goldilocks, the prime `p = 2^64 - 2^32 + 1` (`src/crypto/stark/field/element.rs:19`):

```
  pub const P: u64 = 0xFFFF_FFFF_0000_0001;   // 2^64 - 2^32 + 1
```

`Fp` wraps a `u64` held canonically in `[0, p)`, with addition, subtraction, multiplication,
exponentiation, and inversion. Goldilocks is the standard field for modern STARKs because it is just
under `2^64` (so an element fits a machine word) and its reduction is cheap (`2^64 ≡ 2^32 - 1`, a single
conditional subtraction), and it has a large two-adic subgroup, which is what the FFT-based low-degree
extension needs. All the AIR trace values, constraint evaluations, and FRI codewords live in this
field.

## The hash: an honest description

The proof system's algebraic hash is a Poseidon-*style* permutation over Goldilocks
(`src/crypto/stark/air/poseidon.rs`). It is important to describe it exactly, because it is not a
drop-in of the published reference Poseidon, and overstating that would be wrong. What the code
actually implements is:

```
  width 8, rate 4, capacity 4                 // 256-bit capacity -> ~128-bit sponge security
  S-box x^7                                    // 7 is coprime to the group order, so x^7 is a bijection
  a Cauchy MDS diffusion matrix                // M[i][j] = 1/(x_i - y_j), provably MDS for disjoint nodes
  a full S-box layer every round               // no partial rounds
  round constants by a nothing-up-my-sleeve rule
```

Two of these differ from standard Poseidon and are worth stating plainly. First, the construction uses
a **full S-box layer on every round** rather than the standard Poseidon mix of a few full rounds and
many cheaper partial rounds; this is a more conservative and more uniform round function, at a higher
cost, not the published round schedule. Second, the **round constants are self-derived by a
nothing-up-my-sleeve rule**, the BLAKE3 hash of the domain string `NONOS-POSEIDON-GOLDILOCKS-RC` with
the round and lane indices (`poseidon.rs:224`), rather than the published reference constant set. So
this is a transparent, arithmetization-friendly hash of the Poseidon family; its trustworthiness comes
from the parameters being reproducible and free of hidden structure (anyone can regenerate the
constants from the domain string, and the Cauchy matrix is provably MDS), not from matching a
published parameter set or reference test vectors. There are correspondingly no reference-vector tests
for it, because it is not claiming to reproduce a reference; the guarantee is the auditability of the
derivation.

## Why an algebraic hash

The point of a Goldilocks-native hash is that it can be expressed as field arithmetic, and therefore as
STARK constraints. The `x^7` S-box is a degree-7 polynomial and the MDS mix is linear, so one round of
the permutation is a low-degree constraint over the trace. This is what makes the
[AIR catalog's](air-catalog.md) `Poseidon` and `MerkleMembership` gadgets possible, and it is what lets
the [FRI and transcript](stark.md) run in a Poseidon variant that is itself provable inside the proof
system. A standard byte-oriented hash like BLAKE3 cannot be arithmetized cheaply; this one can, which is
the whole reason it exists alongside the kernel's [general-purpose BLAKE3](../crypto/hashes.md).

## Uses of the permutation

The permutation backs two things (`poseidon.rs:86`, `poseidon.rs:138`): a sponge hash (absorb rate
lanes, permute, squeeze) and a two-to-one Merkle compression (`state = [left | right]`, permute, take
the rate lanes), which is the node hash for the [Poseidon Merkle tree](stark.md) the recursive-friendly
FRI uses.

## Source

```
  src/crypto/stark/field/element.rs   the Goldilocks prime and Fp
  src/crypto/stark/field/            add/sub/mul, exp, inverse
  src/crypto/stark/air/poseidon.rs    the permutation: params, S-box, MDS, NUMS constants, sponge, compress
```

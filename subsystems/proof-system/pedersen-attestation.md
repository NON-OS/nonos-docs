# Pedersen Attestation

The STARK is one proof family; the other is the `zk_kernel`, a set of transparent discrete-log proofs
that back capsule attestation. A capsule can prove its enrolled secret is a member of a committed policy
tree, without revealing the secret, and the kernel checks that proof at spawn. This page documents the
proof construction; the enforcement gate and what the proof binds are on the
[capsule attestation](../../security/attestation.md) page. The code is under `src/crypto/zk_kernel/`.

## The proof family

The `KernelZkVerifier` (`src/crypto/zk_kernel/verifier.rs`) supports a small set of transparent proof
systems:

```
  Range        a committed value lies in a range
  Equality     two commitments hide the same value
  Membership   a committed leaf is in a Merkle tree
  Pedersen     a commitment opens to a claimed value
  Plonk        a general arithmetic-circuit proof
```

Each is transparent: there is no trusted setup and no structured reference string, only public group
elements and hashes. The verifier returns `Valid`, `Invalid`, `MalformedProof`, or
`UnsupportedProofType`, so a garbled proof is rejected as malformed rather than crashing or being
accepted.

## The Pedersen commitment

The base object is a Pedersen commitment over the Curve25519 Edwards group
(`src/crypto/zk_kernel/pedersen.rs`):

```
  commit(value, blinding) = value * G + blinding * H
```

`G` is the standard Ed25519 basepoint. The second generator `H` is the load-bearing detail, and it is
derived so that **no one knows its discrete log with respect to `G`**. `derive_generator_h`
(`pedersen.rs:30`) is a nothing-up-my-sleeve hash-to-curve: it hashes the domain string
`NONOS:TRANSPARENT:PEDERSEN:v1` with `generator_h` and a counter with BLAKE3, tries to decompress the
digest to a curve point, clears the cofactor by multiplying by eight, and takes the first success:

```
  derive_generator_h():
      seed = "NONOS:TRANSPARENT:PEDERSEN:v1" || "generator_h"
      for counter = 0, 1, 2, ...:
          h = blake3(seed || counter)
          if point = decompress(h) exists and 8*point != identity:  return 8*point
```

Because `H` comes out of a hash of a public string, nobody, including whoever wrote the code, knows a
scalar `k` with `H = k*G`. That is what makes the commitment binding without a trusted setup: a prover
who knew `log_G(H)` could open a commitment two ways, and the hash-to-curve derivation rules that out.
The derivation is deterministic, so anyone can recompute `H` and confirm it was not chosen adversarially.

## The membership proof

The enrolled-secret attestation is a membership proof (`src/crypto/zk_kernel/membership.rs`): a Pedersen
commitment to the leaf, a Fiat-Shamir challenge, a Schnorr-style response, and a Merkle path to the
committed root:

```
  prove(leaf, blinding, siblings, directions):
      leaf_commitment = Pedersen.commit(leaf, blinding)
      challenge       = blake3(transcript including leaf_commitment)
      response        = challenge * blinding        // the Schnorr-style response
      -> { path: siblings, directions, leaf_commitment, response }

  verify(root):  recompute the challenge, check the response, walk the path to root
```

The proof convinces the verifier that the prover knows a leaf and a blinding whose commitment sits at a
claimed position in a Merkle tree with the given root, without revealing the leaf. The challenge is the
BLAKE3 hash of the transcript (Fiat-Shamir, making it non-interactive), and the Merkle path binds the
commitment to the committed policy tree. The capsule attestation uses this to prove its enrolled secret
is in the kernel's committed policy tree, bound to the capsule's code, capabilities, and epoch, as the
[attestation gate](../../security/attestation.md) checks.

## The honest caveat: transparent, but classical

Both proof families here are transparent, no trusted setup, but they rest on different assumptions, and
the distinction matters:

- The [STARK, FRI, and Poseidon](stark.md) layer is **hash-based**: its security reduces to the field
  and the hash, with no number-theoretic assumption, which is the conservative, plausibly
  post-quantum foundation.
- This Pedersen attestation layer rests on the **Curve25519 discrete-log assumption** and the random
  oracle (Fiat-Shamir). It is transparent, but it is **classical, not post-quantum**: a large quantum
  computer that breaks discrete log would break the hiding and the soundness of these proofs.

So "transparent" is true of the whole proof system, but "post-quantum" is true only of the STARK layer,
not the Pedersen attestation. This page states that plainly rather than letting "zero-knowledge" and
"transparent" imply a quantum guarantee the discrete-log construction does not provide. The
[attestation verifiers](../../security/attestation.md) are the ones fuzzed against thousands of
adversarial proofs to confirm they reject garbage rather than accept it.

## Source

```
  src/crypto/zk_kernel/pedersen.rs     the commitment and the nothing-up-my-sleeve H
  src/crypto/zk_kernel/membership.rs   the Schnorr-style membership proof + Merkle path
  src/crypto/zk_kernel/equality.rs     the equality proof
  src/crypto/zk_kernel/verifier.rs     KernelZkVerifier and the proof-system enum
  userland/crypto_proofs/src/zk_tests.rs   the adversarial soundness fuzzing
```

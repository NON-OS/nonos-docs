# Developer Roots and Local Builds

The [attestation gate](attestation.md) refuses any capsule it cannot prove is a
member of a policy tree the kernel trusts. That is correct for shipped software,
but it makes building software on NØNOS impossible on its own: a capsule compiled
minutes ago was never enrolled under the vendor's policy tree, so the gate would
refuse it. The answer is not to weaken the gate. It is to let the machine hold
more than one signing authority, to make adding one a deliberate act confirmed on
the trusted path, and to carry through to every attestation which authority
vouched for what. This page documents that mechanism: the authority model, the
per-boot developer-root table, the two-step consent enrolment, and the local-build
identity a machine mints for its own output. The code is under
`src/security/dev_roots/` and `src/security/local_build/`.

## Two authorities, never merged

A successful capsule attestation records not just a measurement but who vouched
for it. `Proved` carries both (`src/security/capsule_attest/proved.rs:25`):

```
  Proved
    measurement  [u8; 32]     what ran, the BLAKE3 the proof was checked against
    authority    Authority    whose word that is
```

`Authority` is a two-variant enum (`src/security/dev_roots/authority.rs:16`):

```
  Authority
    Vendor          the root compiled into the kernel image; cannot be added to at runtime
    Developer(u8)   a key enrolled on this machine by its user, identified by its slot
```

The distinction is never collapsed. A capsule built on this machine and a capsule
shipped by the project are both proved and both run, but they do not mean the same
thing to a remote party deciding whether to trust the machine, so attestation
reports the authority rather than telling that party everything is equally
vendor-signed. `is_vendor()` (`authority.rs:44`) is the predicate a caller uses
where it must not accept locally built code, for example a policy that permits
only vendor capsules to hold hardware capabilities; `Proved::is_vendor`
(`proved.rs:34`) forwards it. The measurement and the authority travel together
everywhere on purpose: a measurement without its authority is a claim that
something was verified without saying against what, which is the exact half-truth
attestation exists to eliminate.

## Where the two roots meet: verify order

`verify_capsule_attestation` (`src/security/capsule_attest/verify.rs:33`) tries
the two authorities in a fixed order, and the order is the security property:

```
  verify_capsule_attestation(trailer, elf, granted_caps):
      vendor = policy_root::root()  or  RootUnavailable
      if against_root::verify(trailer, elf, granted_caps, vendor) ok:
          return Proved { measurement, authority: Vendor }
      for root in enrolled_roots():
          if against_root::verify(trailer, elf, granted_caps, root) ok:
              return Proved { measurement, authority: authority_for(root) }
      return Rejected
```

The vendor root is tried first and always (`verify.rs:38`). Only if that fails are
the enrolled developer roots attempted (`verify.rs:43`), so a capsule that
verifies under the shipped policy is never attributed to a local key, and a local
key can never shadow the vendor's answer. The verification itself is identical
whichever root is used: `against_root::verify` (`against_root.rs:27`) takes the
root as a parameter rather than looking it up, precisely so that a capsule built on
this machine clears exactly the bar a shipped one does, and only membership
differs. On a match under an enrolled root, the reported authority is looked up
from the table by the root's bytes (`authority_for`, `verify.rs:49`), not inferred
from the loop index, so the answer cannot drift if the table is reordered. This is
the same proof and the same 48-byte context binding (ELF hash, granted caps,
policy epoch) documented on the [attestation page](attestation.md); the only thing
the developer-root layer adds is a second set of roots to check membership
against, and a record of which set answered.

## The developer-root table

Enrolled roots live in a small fixed table (`src/security/dev_roots/table.rs`):

```
  MAX_DEV_ROOTS = 4                            (table.rs:11)
  DevRoot { root: [u8; 32], used: bool }
  static TABLE: Mutex<Table>                   (table.rs:60)
```

Four is a deliberate cap, not a resource limit: a machine with dozens of signing
authorities has no meaningful notion of who vouched for what, and enrolment is
meant to be a considered act rather than something that accumulates quietly.
`insert` (`table.rs:29`) returns the existing slot if the same root is already
present, so enrolling a key twice does not consume a second slot, and returns
`None` when the table is full. `enrolled_roots` (`resolve.rs:22`) returns the used
roots by value rather than as a borrow of the table, so the lock is not held while
a caller runs proof verification against them.

The table holds nothing across a reboot. It is an ordinary in-memory `Mutex`, not
persisted, and the module says so as a design statement rather than a limitation
(`mod.rs`): on a system that keeps nothing, a signing authority is not an
exception. `dev_root_count` (`enrol.rs`) is therefore always zero immediately
after boot. A machine re-enrols the authorities it wants each session.

## Enrolment is two steps, gated on the trusted path

Adding a root is split into a request and a confirmation, and neither a capsule
alone can complete (`src/security/dev_roots/enrol.rs`).

`request_dev_root(caller_caps, root)` (`enrol.rs:19`) does not enrol anything. It
checks four preconditions and, if they pass, arms a challenge:

```
  request_dev_root(caller_caps, root):
      if caller_caps lacks EnrolDevRoot         -> Denied
      if !registry_complete()                   -> RegistryIncomplete
      if root == [0u8; 32]                       -> EmptyRoot
      if TABLE.is_full()                         -> NoSlots
      consent::arm_challenge(root)               or EntropyUnavailable
```

The capability check is `caller_caps & Capability::EnrolDevRoot.bit()`
(`enrol.rs:20`), so a capsule can request only if its verified manifest was
granted `EnrolDevRoot`. The `registry_complete()` guard (`enrol.rs:26`) refuses to
widen what the machine will run if the [attestation registry](attestation.md) has
already lost track of what is running: adding an authority whose future output the
machine's own attestation could not describe is exactly the state the registry
exists to prevent. The all-zero-root rejection catches an uninitialised buffer
passed as a root.

`confirm_dev_root(caller_caps, answer)` (`enrol.rs:52`) completes the enrolment
only if `answer` matches the code the challenge displayed. It re-checks
`EnrolDevRoot` rather than assuming it from the request, because the two calls may
arrive from different capsules and a confirmer that could skip the check would only
need to wait for someone else's request to be in flight. On success it inserts the
root and prints `[DEV-ROOT] enrolled slot=<n>; locally built capsules may now run`
(`enrol.rs:63`), returning `Authority::Developer(slot)`.

`request_local_build_root(caller_caps)` (`enrol.rs:40`) is the common case: it does
not let the caller supply the root at all, because a capsule that chose the root
could enrol someone else's tree and then run anything its holder signed. It takes
the root from `local_build::root()` and defers to `request_dev_root`.

### The consent challenge is the whole trust argument

`arm_challenge` (`src/security/dev_roots/consent.rs`) is where the security of
enrolment actually rests, and it rests on one property: the confirmation code is
written by the kernel straight to the serial console, which no capsule can read.

```
  CHALLENGE_MODULUS = 1_000_000                 (consent.rs)  six digits
  arm_challenge(root):
      challenge = random_u32_secure() % CHALLENGE_MODULUS   or None
      PENDING.arm(root, challenge)
      print to the console:
        "== DEVELOPER ROOT ENROLMENT REQUESTED =="
        "Software built on this machine wants permission to run here."
        "Confirmation code: <challenge>"
```

A capsule may ask to enrol a root, but it cannot learn the six-digit number needed
to complete it, so the confirmation can only come from somebody physically looking
at the machine and typing the code. The challenge comes from
`crypto::rng::random_u32_secure`; when the entropy source will not answer, the
function returns `None` rather than falling back to a constant, because a constant
would be a code the caller could guess, and the caller is the one asking to be
approved. `redeem(answer)` (`consent.rs`) completes the pending enrolment only on a
match. The comment states the design rationale plainly: weaker designs put the
decision in a window drawn by userspace, which only moves the trust into whichever
process draws the window, so a compromised shell would approve silently on the
user's behalf. Writing the code to a surface no capsule can read is what keeps the
decision with the human.

The two syscalls that expose this to userspace are `MkDevRootRequest` and
`MkDevRootConfirm` (`src/syscall/microkernel/enrol_dev_root.rs:17`), which call
`request_dev_root` and `confirm_dev_root` respectively; the capability gate is
inside those functions, not in the syscall capability table.

## The local-build identity

For the machine to run what it builds, each local build needs a membership proof
under a root the machine will accept. `src/security/local_build/` mints that
identity (`local_build/mod.rs`): one secret, one leaf, one root, held for the life
of the boot.

`LocalIdentity` (`local_build/identity.rs:23`) is a Pedersen commitment to a
freshly drawn secret, folded into a one-leaf tree root:

```
  mint():
      secret   = get_random_bytes_secure()      or None
      blinding = get_random_bytes_secure()      or None
      commitment = PedersenCommitment::commit(secret, blinding)
      root       = root_for(commitment)
```

The secret is drawn from the secure RNG rather than a best-effort source
(`identity.rs:34`), because a guessable secret is a tree anyone could mint proofs
against. `root()` (`identity.rs:45`) returns the identity's root and is stable for
the life of the boot, so a second build does not invalidate the consent given for
the first. The membership proof each build carries binds the measurement and the
capabilities into its challenge (`local_build/mod.rs`), so a trailer minted for a
capsule holding nothing does not verify for the same bytes installed with a wider
grant. Minting a proof is explicitly not consent: `local_build` enrols nothing,
and a build's proof is worthless until its root is enrolled through the
trusted-path flow above.

## Status

| Mechanism | Status |
|-----------|--------|
| Vendor-first verify order (`verify.rs:38-53`) | IMPLEMENTED, ENFORCED |
| `Authority` recorded on every `Proved` (`proved.rs`, `verify.rs`) | IMPLEMENTED, ENFORCED |
| Two-step consent enrolment, capability-gated (`enrol.rs`) | IMPLEMENTED, ENFORCED |
| Trusted-path challenge on the serial console (`consent.rs`) | IMPLEMENTED, ENFORCED |
| Table bounded at 4, deduplicating (`table.rs`) | IMPLEMENTED, ENFORCED |
| No persistence across reboot (in-memory `Mutex`) | IMPLEMENTED (by construction) |
| Local-build identity from secure RNG (`identity.rs`) | IMPLEMENTED |
| Adversarial or unit tests for this subtree | NOT ESTABLISHED from repo |

The one honest boundary: this subtree carries no in-tree test that exercises the
enrolment guards or the verify order under attack. The properties above are read
from the code, not from a passing negative-test suite. The membership proof the
roots gate is the same one the [attestation](attestation.md) and
[proof-system](../subsystems/proof-system/stark.md) pages cover, and the
fuzzing that exists is on that verifier, not on this enrolment path.

## Source map

```
  src/security/dev_roots/authority.rs   Authority { Vendor, Developer(u8) }, is_vendor
  src/security/dev_roots/table.rs       the bounded, deduplicating root table
  src/security/dev_roots/resolve.rs     authority_for, enrolled_roots
  src/security/dev_roots/enrol.rs       request_dev_root, confirm_dev_root, request_local_build_root
  src/security/dev_roots/consent.rs     arm_challenge, redeem, the six-digit console code
  src/security/dev_roots/pending.rs     the pending-enrolment cell
  src/security/local_build/identity.rs  the per-boot local-build secret, commitment, root
  src/security/local_build/mod.rs       proof minting bound to measurement and caps
  src/security/capsule_attest/verify.rs the vendor-first, then-enrolled verify order
  src/security/capsule_attest/proved.rs Proved { measurement, authority }
  src/syscall/microkernel/enrol_dev_root.rs  MkDevRootRequest / MkDevRootConfirm
```

The gate that consumes `Proved` and prints the authority is on the
[attestation](attestation.md) page; the capability that gates enrolment is one of
the bits on the [capabilities](capabilities-and-tokens.md) page.

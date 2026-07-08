# capsule_attest

`capsule_attest` answers questions about the system's identity and its stated invariants: a health
check, a product summary, the boot identity, the invariant list, and the capsule list. Its name invites
an assumption the code does not support, and this page is careful about the distinction: the capsule
returns authored, human-readable statements about the system and a boot label, not cryptographic proofs
computed at request time. The statements are true and their cited mechanisms are real kernel machinery,
but they are a signed-off audit manifest, not proof objects. Service `attest` on port 4444, capability
mask `0x19`. The source is `userland/capsule_attest/`.

## Contents

- [The server loop](#the-server-loop)
- [The operations](#the-operations)
- [What the responses actually are](#what-the-responses-actually-are)
- [The invariants, in full](#the-invariants-in-full)
- [The boot identity](#the-boot-identity)
- [Where the real proofs live](#where-the-real-proofs-live)
- [Honest scope](#honest-scope)
- [Source map](#source-map)

## The server loop

`main.rs:29` initializes the heap and calls `server::run` (`src/server/runner.rs:27`), a stateless
request loop with two 64 KiB buffers that replies directly to the attested sender:

```
  run():
      loop:
          n = mk_ipc_recv_from(SERVICE_PORT 4444, in_buf, &sender_pid)
          if n <= 0 or sender_pid == 0:  yield; continue
          m = route(in_buf[..n], out_buf)
          mk_ipc_reply(sender_pid, out_buf[..m])
```

The frame is magic `0x41545354` ("ATST"), version 1, with an op, flags, and request id; there is no
request payload, every response is generated server-side.

## The operations

Five read-only operations (`src/protocol/ops.rs:17`):

```
  1  HEALTHCHECK    2  PROOF_SUMMARY    3  PROOF_INVARIANTS    4  PROOF_BOOT    5  PROOF_CAPSULE_LIST
```

`route` (`src/handlers/router.rs:24`) validates the header (a bad magic or version is rejected before
routing) and dispatches; an unknown op returns an error status.

## What the responses actually are

Precisely: the `PROOF_*` operations return pre-authored data, not proof computations.

- **PROOF_SUMMARY** (`src/handlers/proof_summary.rs:21`) writes length-prefixed product fields, the name,
  tagline, and version, from static constants.
- **PROOF_INVARIANTS** (`src/state/invariants.rs:23`) returns a static array of six invariants, each a
  `name`, a `claim`, and a `mechanism`, all authored as constant byte strings.
- **PROOF_BOOT** (`src/handlers/proof_boot.rs`) returns a timestamp and a label.
- **PROOF_CAPSULE_LIST** returns the capsule listing.

There is no STARK, no zk_kernel, and no signature computed in these handlers.

## The invariants, in full

The six invariants (`src/state/invariants.rs:23`) are the heart of the response, and they are worth
reproducing because each `mechanism` names a real kernel component documented elsewhere in this wiki:

```
  NO LOGS
    claim:     no shipped capsule may emit MkDebug or open a serial surface
    mechanism: every shipped Capsule.mk has the Debug bit absent from CAPSULE_REQUIRED_CAPS;
               the kernel rejects MkDebug syscalls outside the mask

  NO TRACES
    claim:     no persistent user identifier or content survives a capsule exit
    mechanism: capsules refuse the FileSystem cap unless granted; the clipboard has idle auto-clear;
               the input_router holds no history

  EPHEMERAL
    claim:     all state is RAM-resident; no on-disk record unless a capsule declares FileSystem
    mechanism: only ramfs + vfs touch disk surfaces; the trust keystore is read-only at boot

  NOT LINUX
    claim:     no POSIX shapes, no errno tables, no fd numbering, no signal model
    mechanism: the Mk* 4-byte ASCII tag syscall ABI; an NCMP-style wire across every capsule;
               a NONOS-native capability taxonomy

  PRIVACY MICROKERNEL
    claim:     every capsule runs CPL=3 with a static capability mask the kernel enforces at every syscall
    mechanism: capsule_spawn::spawn_verified records the caps_bits; syscall dispatch checks the mask
               before every routed handler; the mask is signed in the capsule manifest

  HYBRID-PQ SIGNATURES
    claim:     every binary loaded at runtime is signed Ed25519 + ML-DSA-65 and chains to the anchor
    mechanism: capsule_spawn::spawn_verified rejects any ELF whose nonos_id_cert + manifest do not both
               verify against BAKED_TRUST_ANCHOR_POLICY
```

Each mechanism corresponds to code this wiki documents: the capability-mask enforcement is the
[syscall contract](../../subsystems/syscall/boundary.md), the hybrid signatures are the
[verified-spawn gate](../../security/capsules-and-trust.md) requiring both Ed25519 and ML-DSA-65, the
RAM-residency is the [zeroization](../../subsystems/memory/zeroization.md) posture, and the
`Mk*` ABI is the [syscall numbers](../../subsystems/syscall/numbers.md). So the invariants are an accurate
catalogue of guarantees, and each is checkable against the cited code, they are just not proven by this
capsule at request time.

## The boot identity

`PROOF_BOOT` (`src/handlers/proof_boot.rs`) returns the current time and a fixed label,
`"NONOS bootloader (hybrid Ed25519 + ML-DSA-65)"`. It is a boot-identity string plus a timestamp, not a
cryptographic attestation chain. The actual boot attestation, the kernel's measured `zk_verified` and
`secure_boot` status, is read by the [boot splash](boot-splash.md) directly from the kernel through
`mk_attest_status`, not from this capsule.

## Where the real proofs live

The genuine cryptographic proof machinery is in the kernel, not in this capsule: the transparent
[STARK and Pedersen attestation](../../subsystems/proof-system/README.md), and the
[capsule attestation gate](../../security/attestation.md) that verifies a capsule's enrolled-secret proof
at spawn. This capsule is a system-information and stated-invariant service, and documenting it as such,
rather than as an attestation engine, is the honest reading of its code.

## Honest scope

The capsule is a truthful audit surface: its invariants describe real, code-backed guarantees, and its
summary and boot responses are accurate. What it is not is a producer of cryptographic proofs; the
`PROOF_*` names describe the *subject* (the system's proofs and properties) rather than a proof computed
on demand. It has no per-caller state, no key material, and no dependency on the STARK or zk_kernel code.

## Source map

```
  userland/capsule_attest/src/server/runner.rs      the reply-to-sender loop
  userland/capsule_attest/src/handlers/router.rs     op routing + header validation
  userland/capsule_attest/src/handlers/proof_summary.rs, proof_boot.rs, proof_capsule_list.rs
  userland/capsule_attest/src/state/invariants.rs    the six authored invariants (name/claim/mechanism)
  userland/capsule_attest/src/state/product.rs       the product name/tagline/version constants
```

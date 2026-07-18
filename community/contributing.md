# Contributing

NØNOS is built by people who would rather prove a thing than assert it. If that
is you, here is where to start.

## Get the tree building

Clone with submodules, then build and boot the full system once so you know your
environment is good:

```sh
git clone --recursive https://github.com/NON-OS/nonos-micro-kernel.git
cd nonos-micro-kernel
make qemu
```

The [quickstart](../build/quickstart.md) has the toolchain and QEMU details. The
first build is long; everything after is incremental.

## Where to work

Write a capsule. This is the most common contribution and the most
self-contained. A capsule is a signed, sandboxed program that talks to the
kernel over IPC and carries a proof of what it is. The
[quickstart](../build/quickstart.md) walks the whole path from a clean clone to
your capsule attested and running on the desktop, and
[verifiable-work.md](verifiable-work.md) covers the receipt lane on top of it.

Work on the kernel. Primitives only: memory, scheduling, IPC, capabilities, the
crypto floor, the spawn gate. Policy belongs in a capsule, not here. Read the
[architecture overview](../architecture/overview.md) first so a change lands on
the right side of that line.

Strengthen a proof. The attestation and the core algorithms are checked by Lean,
Verus, Kani and runnable proof crates under `verification/` and `*_proofs/`. A
new theorem, a tighter bound, or a harness that finds a real bug is as welcome as
code. The [verification page](../architecture/verification.md) states exactly
what is proved today.

Bring up hardware. A driver is a capsule. The register facts it needs are
hardware, not license-encumbered source; reimplement rather than copy. Existing
drivers under `userland/capsule_driver_*` are the pattern.

## House rules

Nothing merges that cannot build and prove itself. CI runs the full trust chain,
the proof tools and a boot. Keep unsafe minimal and justified. No panics on the
kernel path; handle the error. Run `cargo fmt` and `cargo clippy` before you
push. Match the surrounding code: its naming, its comment density, its idiom.

## Reporting a weakness

Security issues go through [private reporting](../security/reporting.md), not a
public issue. It sets out scope, disclosure and the trusted computing base.

## What it pays

Securing NØNOS is rewarded. [rewards.md](rewards.md) has the detail.

# The Boot Handoff

The kernel does not run first. A separate signed bootloader verifies the kernel image, sets up the
initial environment, and hands the kernel a structure describing the machine. This page documents that
handoff and the verification in front of it. The code is under `src/boot/handoff/`, and the bootloader
is the separate `nonos-bootloader` crate.

## The handoff structure

The bootloader passes a `KernelHandoff` (`src/boot/handoff/kernel_handoff/handoff.rs:26`):

```
  struct KernelHandoff {
      memory:      MemoryHandoff,       // the firmware memory map
      framebuffer: Option<Framebuffer>, // the GOP framebuffer, if any
      timing:      TimingHandoff,       // TSC frequency + Unix epoch at boot
      firmware:    FirmwareHandoff,     // ACPI / SMBIOS pointers
      arch:        ArchSpecificHandoff, // the per-ISA payload (X86_64 { v1 })
  }
```

This is the contract between the bootloader and the kernel: the memory map the [frame allocator](../memory/physical-frames.md)
seeds from, the framebuffer the [display](../graphics/README.md) maps, the timing the
[clock](../time-and-clock/calibration.md) uses to skip re-calibrating what the bootloader already
measured, and the firmware tables. The `arch` field is the `ArchSpecificHandoff` enum
(`src/boot/handoff/kernel_handoff/arch.rs:24`); it carries an `X86_64` arm (built from the loader's
`BootHandoffV1`, `.../x86_64/from.rs:29`) and an `Aarch64` arm (built from the device-tree `BootInfo`,
`.../aarch64/from.rs:27`), which is the same [multi-architecture](../smp/README.md) boundary the rest of
the kernel uses. The kernel downcasts the arch field once, in the `init_arch_*` helpers, keeping the
ISA-specific handoff handling in one place.

## Verification before handoff

The handoff is only reached after the bootloader has verified the kernel it is about to run. The
bootloader authenticates the kernel image with the same hybrid signature posture as a
[capsule](../../security/capsules-and-trust.md), an Ed25519 and an ML-DSA-65 signature, and it enforces
an anti-rollback index against a TPM monotonic counter so an attacker cannot downgrade the kernel to an
older, vulnerable signed image.

With the default `stark-kernel-attest` feature the bootloader adds a third check on top of the signature
and the rollback index: it verifies the kernel's own transparent STARK self-attestation before the jump.
It measures the kernel image with BLAKE3 and checks the proof, carried in the image trailer, that this
measurement is a leaf of the enrolled kernel root
(`nonos-bootloader/src/kernel_verify/stark_attest.rs:43`). The proof is verified by the same `nonos-stark`
crate the kernel links, so the prover and the verifier are one implementation. A zeroed root trusts
nothing, so an un-enrolled build cannot be spoofed. Only a kernel that passes the signature check, the
rollback check, and, when enabled, the self-attestation, is executed and handed the structure above. The
full loader pipeline is documented on the [bootloader](bootloader.md) page.

This means the trust chain is unbroken from the
firmware root to the running capsules: the bootloader vouches for the kernel, the baked trust anchor in
the kernel vouches for the capsule certificates, and each capsule's manifest vouches for its image.

## The security handoff

The handoff also carries the boot-time security context (`src/boot/handoff/types/security.rs`) the
bootloader established, so the measurements and state the bootloader produced are available to the
kernel rather than being re-derived. The bootloader is documented in its own tree; from the kernel's
side, what matters is that by the time `microkernel_init` runs, the image has already been
authenticated and the environment described. The detailed bootloader hardening, the TPM counter, and
the hybrid kernel-signature scheme live with the bootloader crate and the
[security](../../security/trust-anchor.md) section.

## Security analysis

The handoff is the seam where trust crosses from the bootloader into the kernel, so the properties
that matter here are about what has already been proven by the time the kernel is handed the structure,
not about the structure itself.

**Fail-closed rollback in front of the handoff.** The bootloader's `check_rollback`
(`nonos-bootloader/src/boot/crypto/rollback/check.rs:28`) gates on the signed `rollback_index`
(`check.rs:42`) and compares it against both the recorded kernel version (`check_kernel_version`,
`check.rs:43`) and the TPM monotonic floor (`read_floor`, `check.rs:64`). When the mode requires a
signature and either check fails, it calls `fatal_reset` (`check.rs:59`, `check.rs:73`), so a downgraded
image does not reach the handoff at all. The floor is committed forward on a good boot by
`commit_rollback` (`nonos-bootloader/src/boot/crypto/rollback/commit.rs:28`), which pushes the index into
the TPM counter through `commit_floor` (`commit.rs:61`). The honest boundary the code records: when the
mode does not require a signature, a detected rollback logs "rollback detected but dev mode - continuing"
(`check.rs:61`) and proceeds, so the anti-rollback guarantee holds only for signature-enforcing boots.

**The trust chain is unbroken by construction.** The bootloader authenticates the kernel before it
jumps to it, so the running kernel is vouched for by the firmware-rooted bootloader. The kernel in turn
carries a baked trust anchor (decoded at each capsule spawn in the mirror's
`src/userspace/capsule_*/spawn.rs`) that every capsule certificate is checked against. Nothing in the
handoff structure is trusted as authority: it is a description of the machine (memory map, framebuffer,
timing, CPU topology), and the security-relevant decisions were already made before it was populated.

## Debugging the boot handoff

The machine this runs on may have no serial port, and the kernel's on-screen text console is off by
design (see [kernel init](kernel-init.md)), so the primary handoff diagnostic is the bootloader's own
log, which draws to the GOP framebuffer before the kernel takes over. On a signature-enforcing boot the
lines to look for are "Anti-rollback check PASSED" (`check.rs:47`) and the
"tpm floor {} index {}" info line (`check.rs:65`), which print the floor the TPM counter enforces and
the index the image carries. A boot that dead-ends on the bootloader's red error screen
(`show_error_screen`, `check.rs:57` / `check.rs:71`) is the footer version or the TPM floor rejecting the
image, not a kernel fault. If the footer itself will not parse, `commit_rollback` logs "kernel version
footer parse failed" (`commit.rs:36`) and resets. When the bootloader log shows the image accepted but
the kernel never prints its own first serial line, the handoff structure reached the kernel and the fault
is in early kernel bring-up rather than verification.

## Source map

```
  src/boot/handoff/kernel_handoff/handoff.rs        the KernelHandoff structure (memory, cpus, console,
                                                    framebuffer, timing, measurement, arch)
  src/boot/handoff/kernel_handoff/arch.rs           the ArchSpecificHandoff enum (X86_64, Aarch64 arms)
  src/boot/handoff/types/                            the handoff payload types
  src/boot/firmware.rs                               the ACPI / SMBIOS init from the handoff
  nonos-bootloader/src/boot/crypto/rollback/         check_rollback / commit_rollback, the TPM floor
  nonos-bootloader/                                  the bootloader crate (kernel signature verification)
```

Every reference above is verified against those trees. The init sequence the kernel runs once it holds
this structure is on the [kernel init](kernel-init.md) page; the baked trust anchor the kernel roots
capsule trust in is in the [security](../../security/trust-anchor.md) section.

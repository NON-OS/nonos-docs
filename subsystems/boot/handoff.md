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
measured, and the firmware tables. The `arch` field is an enum whose only arm today is `X86_64`,
carrying the arch-specific handoff; other architectures add arms as their boot trees land, which is
the same [multi-architecture](../smp/README.md) boundary the rest of the kernel uses. The kernel
downcasts the arch field once, in the three `init_arch_*` helpers, keeping the ISA-specific handoff
handling in one place.

## Verification before handoff

The handoff is only reached after the bootloader has verified the kernel it is about to run. The
bootloader authenticates the kernel image with the same hybrid signature posture as a
[capsule](../../security/capsules-and-trust.md), an Ed25519 and an ML-DSA-65 signature, and it enforces
an anti-rollback index against a TPM monotonic counter so an attacker cannot downgrade the kernel to an
older, vulnerable signed image. Only a kernel that passes both the signature check and the rollback
check is executed and handed the structure above. This means the trust chain is unbroken from the
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

## Source

```
  src/boot/handoff/kernel_handoff/handoff.rs   the KernelHandoff structure
  src/boot/handoff/types/                       memory, framebuffer, timing, firmware, security
  src/boot/firmware/                            the ACPI / SMBIOS init from the handoff
  nonos-bootloader/                             the bootloader crate (verification, TPM rollback)
```

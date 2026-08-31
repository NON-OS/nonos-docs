# The Bootloader and the Verified-Boot Pipeline

The kernel does not run first. On x86_64 a separate signed UEFI application, the `nonos-bootloader`
crate, runs under firmware, verifies the kernel image, establishes the boot-time security context, and
only then builds the [handoff structure](handoff.md) and jumps to the kernel. This page documents that
pipeline stage by stage, and where trust and measurement sit in it. The code is the `nonos-bootloader`
crate; the entry is `nonos-bootloader/src/main.rs:27` (`efi_main`).

The aarch64 boot path is different and does not use this loader: firmware enters the kernel's own
`_start` (`src/arch/aarch64/asm/start.S:13`) with a device-tree pointer, which drops to EL1 and calls
`kernel_entry` (`src/arch/aarch64/boot/entry.rs:32`). That path is covered in [kernel init](kernel-init.md);
this page is the x86_64 UEFI loader.

## The two phases

`efi_main` calls `boot_entry` (`nonos-bootloader/src/entry/boot.rs:28`), which runs a pre-verification
phase, then hands off to `run_verified_boot` (`nonos-bootloader/src/entry/pipeline.rs:28`), the
verify-and-jump phase. The re-exported stage functions live under `nonos-bootloader/src/boot/`
(`nonos-bootloader/src/boot/mod.rs`).

```
  boot_entry                          entry/boot.rs:28
    run_uefi_init                     boot/uefi/init.rs:27      GOP, logger, config, firmware quirks
    dev_override                      entry/dev.rs              developer-mode key check
    run_security_checks               boot/security/run.rs:33   posture: canaries, HW reqs, platform, TPM self-test
    run_hardware_discovery            boot/hardware.rs:24       ACPI RSDP / PCI / memory map
    select_security_mode              entry/mode.rs             pick the enforcing/development mode
    enforce_policy                    boot/security/policy.rs:28  fail closed if the mode forbids boot
    initialize_zk_replay_protection   boot/zk_init/init.rs:24   boot nonce + machine id; secure-halt if unset
        |
        v
  run_verified_boot                   entry/pipeline.rs:28
    run_kernel_load                   boot/kernel/load.rs:30    read kernel.bin from the ESP
    run_crypto_verification           boot/crypto/run.rs:30     BLAKE3 measure, dual signature, rollback check
    attest_kernel                     boot/attestation/kernel_gate.rs:37   STARK self-attestation gate
    run_elf_parse                     boot/elf/parse.rs         parse the kernel ELF
    commit_rollback                   boot/crypto/rollback/commit.rs:28    advance the TPM anti-rollback floor
    run_handoff_prepare               boot/prepare/run.rs:35    build BootHandoffV1, exit boot services, jump
```

Every step before `run_kernel_load` is environment and posture; the trust decisions are in the verify
phase. The pipeline is `-> !`: it either jumps to a verified kernel or resets the machine.

## Loading the kernel image

`run_kernel_load` (`boot/kernel/load.rs:30`) reads the kernel off the EFI System Partition through
`SimpleFileSystem`, trying `\EFI\nonos\kernel.bin`, then `\EFI\nonos\nonos_kernel`, then `\kernel.bin`
(`nonos-bootloader/src/loader/file/load.rs:73` onward). A missing image is a `fatal_reset`, not a
fallback: there is nothing to verify, so there is nothing to boot. This is the file the build wrote as
`kernel_attested.bin` and the ESP packaging copied to `EFI/nonos/kernel.bin` (`mk/20-build.mk:1148`).

## Trust point 1: measure and verify the dual signature

`run_crypto_verification` (`boot/crypto/run.rs:30`) first measures the loaded image with BLAKE3
(`boot/crypto/hash.rs:29`), recording the measurement, then verifies the signature. The real
verification is `verify_kernel_crypto` (`nonos-bootloader/src/kernel_verify/verify.rs:31`). The signed
message is `hash || rollback_index` (`kernel_verify/signature_message.rs:17`), so the rollback index is
inside the signature, not a mutable field beside it.

The production posture is a hybrid signature, verified in `kernel_verify/signature_hybrid.rs:27`: an
Ed25519 check (`signature_ed25519.rs:35`) **and** an ML-DSA-65 check (`signature_hybrid.rs:44`), and
`signature_valid` is the conjunction of the two (`signature_hybrid.rs:41`). A break in one algorithm
does not by itself admit a forged kernel. An all-zero signature is rejected as unsigned
(`kernel_verify/signature.rs:37`), and the signer identity is bound by a constant-time key-id compare of
both signers plus a release magic (`kernel_verify/signature_policy.rs:29`).

## Trust point 2: the STARK self-attestation gate

With the default `stark-kernel-attest` feature (on unless `NONOS_STARK_KERNEL_ATTEST := 0`, see
`mk/00-config.mk:154`), `attest_kernel` (`boot/attestation/kernel_gate.rs:37`) runs the STARK gate
(`stark_kernel_gate`, `boot/attestation/kernel_gate.rs:61`). The proof itself is checked during crypto verify by
`verify_kernel_stark_self_attestation` (`kernel_verify/verify.rs:84`, called from `verify.rs:75`), which requires the trailer magic
`NZKSTRK1` and calls `stark_attest::verify_kernel_self_attestation`
(`kernel_verify/stark_attest.rs:43`). That routine refuses an all-zero (unenrolled) root
(`stark_attest.rs:48`), binds the proof to `BLAKE3(kernel_bytes) || BOOT_EPOCH` (`stark_attest.rs:51`),
and verifies a money-grade membership trailer against the enrolled `KERNEL_ATTEST_ROOT` baked in at the
loader's build time. The gate verdict is fail-closed in production: a missing or invalid proof is a
`fatal_reset` (`boot/attestation/kernel_gate.rs:99`); only Development mode may skip it with a warning.

The prover and the verifier are the same `nonos-stark` implementation the kernel and the enrollment tool
link, so there is one membership check, not two that must be kept in agreement.

## Trust point 3: the TPM anti-rollback floor

The anti-rollback check runs inside `run_crypto_verification` before commit, in
`check_rollback` (`boot/crypto/rollback/check.rs:28`). It reads the signed `rollback_index`, compares it
against the NVRAM kernel-version record (`check_kernel_version`), reads the TPM monotonic counter floor
(`read_floor`, `check.rs:64`), and, in an enforcing mode, calls `fatal_reset` if the image index is below
the floor (`check.rs:73`). The honest boundary the code records: in a non-signature-enforcing (dev) mode
a detected rollback logs `rollback detected but dev mode - continuing` (`check.rs:61`) and proceeds, so
the anti-rollback guarantee holds only for signature-enforcing boots.

After the kernel passes every check, `commit_rollback` (`boot/crypto/rollback/commit.rs:28`) advances the
floor: it updates the NVRAM version (`commit.rs:47`) and commits the signed index into the TPM monotonic
counter (`commit_floor`, `commit.rs:61`). An older signed kernel cannot then be rolled back onto the
device. Note the anti-rollback floor ratchets and is enforced by hardware; the mechanics are in
`security/tpm_nv/floor.rs`.

## What the TPM does and does not measure on the ship path

Being precise here matters, because the shipping profile and the legacy profile differ. During
`run_security_checks` the loader extends **PCR8** (`PCR_BOOTLOADER`) with a probe value while detecting
measured boot (`security/check/tpm.rs:23`), and it runs a TPM monotonic-counter self-test
(`boot/security/run.rs:41`). The kernel-hash PCR extension (**PCR9** with `kernel_hash || zk_proof_hash`,
**PCR14** with the proof hash) lives in `extend_boot_measurements`
(`security/enforce/checks/measurements.rs:29`), but that path is reached only from the **legacy** ZK
success path (`boot/attestation/run/success.rs:42`). The default `stark-kernel-attest` build does not
take that path: on the ship image the kernel's binding to the TPM is the **monotonic anti-rollback
floor** (trust point 3) plus the STARK membership proof (trust point 2), not a PCR extend of the kernel
hash. The kernel itself performs no TPM PCR extend during its own init (see [kernel init](kernel-init.md));
the kernel-side `verify_kernel_self_attestation` twin (`src/security/kernel_attest.rs:40`) has no caller
in `src/`, and the kernel reaches the TPM only later and on demand, through the `sys_attest_doc` syscall
(`src/syscall/microkernel/attest_doc.rs:35`).

## Building the handoff and the jump

`run_handoff_prepare` (`boot/prepare/run.rs:35`) collects entropy, builds the crypto and firmware
handoff, generates a boot attestation quote, seals the audit log, and calls `exit_and_jump`
(`handoff/exit/orchestrate.rs:33`). That routine allocates and fills the `BootHandoffV1`
(`handoff/exit/handoff_init.rs:27`): magic `0x4E4F4E4F` and version 1 (`handoff/types/constants.rs:17`),
the entry point, framebuffer, ACPI and SMBIOS pointers, timing, and the measurement block
(`init_measurements`, `handoff_init.rs:50`: the kernel BLAKE3, the signature-valid flag, secure-boot, and
the attestation-ok flag). It then calls `exit_boot_services` (`orchestrate.rs:90`), copies the final
memory map, switches CR3 to the kernel PML4 (`orchestrate.rs:103`), and jumps
(`validate_and_jump` -> `handoff/jump/entry.rs:27`, arch asm in `arch/x86_64/asm/handoff_jump.S`). The
kernel lands in `kernel_entry` (`src/nonos_main.rs:52`) with `rdi = handoff_ptr`.

The kernel does not re-derive any of the security-relevant decisions. By the time it runs, the image has
been measured, dual-signature-verified, STARK-attested, and rollback-checked; the handoff is a
*description of the machine* plus the *record of what was already proven*, and the kernel reads those
measurements (`src/entry/security.rs:19`) rather than trusting the handoff as an authority.

## Reproducibility

The loader build is reproducible on purpose: `make nonos-mk-verify-reproducible-boot`
(`mk/20-build.mk:145`) builds `nonos_boot.efi` twice from identical inputs and asserts the two outputs are
byte-identical, so anyone can rebuild the loader from public source and confirm a published hash.
`SOURCE_DATE_EPOCH` is pinned by the Makefile (`mk/00-config.mk:12`).

## Source map

```
  nonos-bootloader/src/main.rs                       efi_main, the UEFI entry
  nonos-bootloader/src/entry/boot.rs                 boot_entry, the pre-verification phase
  nonos-bootloader/src/entry/pipeline.rs             run_verified_boot, the verify-and-jump phase
  nonos-bootloader/src/boot/kernel/load.rs           kernel image load from the ESP
  nonos-bootloader/src/boot/crypto/                  BLAKE3 measure, signature, rollback
  nonos-bootloader/src/kernel_verify/                the dual-signature and STARK verification
  nonos-bootloader/src/boot/crypto/rollback/         check_rollback, commit_rollback, the TPM floor
  nonos-bootloader/src/boot/attestation/kernel_gate.rs  the STARK self-attestation gate
  nonos-bootloader/src/boot/prepare/ , handoff/exit/    build BootHandoffV1, exit boot services, jump
```

Every reference is verified against those trees. The structure the loader builds and the kernel consumes
is on the [handoff](handoff.md) page; the init sequence the kernel runs once it holds it is on the
[kernel init](kernel-init.md) page.

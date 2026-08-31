# Kernel Initialization

When the bootloader hands control to the kernel, the kernel brings the system up in a fixed order, each
step a precondition for the next, and any step that cannot complete is fatal. This page walks that
sequence for x86_64, and notes where aarch64 converges onto the same shared path. The init code is under
`src/kernel_core/init/entry/`.

## Arch entry, before the shared init

On x86_64 the bootloader jumps to `kernel_entry` (`src/nonos_main.rs:52`) with `rdi = handoff_ptr`. That
function validates the handoff, then runs `init_core_systems` (`src/nonos_main.rs:91`) and finally
`boot_microkernel` (`src/nonos_main.rs:98`), which calls `microkernel_init` (`nonos_main.rs:104`) and
`microkernel_main` (`nonos_main.rs:106`).

`init_core_systems` (`src/boot/main/core_init/init_core_systems.rs:25`) is the x86-specific pre-init that
brings up the machinery `microkernel_init` assumes:

```
  serial console                      init_core_systems.rs:26
  boot timer / TSC anchor             :28, :32, :33
  GDT                                 :35
  SYSCALL MSRs (STAR/LSTAR/CSTAR)     :40
  early IDT, then full IDT            :48, :53
  bootstrap heap / global allocator   :51
  ACPI tables (RSDP from handoff)     :56
  LAPIC init, idle + preemption timer :57, :62, :63
  sti (interrupts on)                 :73
  memory encryption, PCI, IOMMU       :76, :77, :83
```

On aarch64 there is no UEFI loader and no `init_core_systems`. Firmware enters `_start`
(`src/arch/aarch64/asm/start.S:13`) with `x0 = dtb_ptr`, drops from EL2 to EL1, zeroes BSS, and calls
`kernel_entry` (`src/arch/aarch64/boot/entry.rs:32`), which brings up the CPU, UART, MMU, vector table,
GIC and timer (`entry.rs:38`-`55`), parses the device tree, builds a `KernelHandoff`, and then calls the
same `microkernel_init` (`entry.rs:74`). From `microkernel_init` on, the two architectures run identical
code.

## The shared sequence: microkernel_init

`microkernel_init(&KernelHandoff)` (`src/kernel_core/init/entry/microkernel_init.rs:30`) runs the
bring-up in order. The steps are grouped into focused sub-init functions, each in its own file under
`src/kernel_core/init/entry/`:

```
  1.  init_arch_memory_and_framebuffer   :32   EFI/DTB memory map -> frame allocator; early framebuffer
  2.  boot log init after framebuffer    :33
  3.  init_arch_firmware                  :37   ACPI / SMBIOS (x86); no-op on aarch64 (DTB already parsed)
  4.  init_core_services                  :38   RNG, IPC secret, SMP BSP, scheduler, clock
  5.  init_vm_and_protection              :39   unified VM / paging, MMU, WP/SMAP/NX, stack guards
  6.  init_platform_baseline              :45   remap PCI windows, PCI, hardware broker, entropy,
                                                 boot-session nonce, token signing key
  7.  init_arch_framebuffer               :55   MMIO-map the framebuffer (needs paging)
  8.  init_device_routing                 :57   cache boot CPU id, IO-APIC / GIC IRQ routing
  9.  init_process_runtime                :58   process tables, ELF loader, kernel keys
  10. start_secondary_cpus                :66   the application processors
```

The fatal, security-critical steps live inside step 4, `init_core_services`
(`src/kernel_core/init/entry/init_core_services.rs:25`): RNG init (`:34`), the IPC MAC secret (`:37`),
the SMP bootstrap-processor init (`:40`), the scheduler (`sched::init`, `:43`), and the clock (`:46`).
Step 5, `init_vm_and_protection` (`init_vm_and_protection.rs:23`), brings up unified paging
(`init_unified_vm`, `:27`). Step 6, `init_platform_baseline` (`platform/baseline.rs:35`), seeds the
hardware broker, entropy, the boot-session nonce (`:59`), and the capability-token signing key (`:60`).
Step 9, `init_process_runtime` (`init_runtime.rs:39`), brings up process management (`:40`), the ELF
loader (`:41`), and the kernel Ed25519 keypair (`:42`).

## Why the order

The dependencies are load-bearing, and the grouping reflects them:

- **RNG before the IPC secret.** The [IPC MAC key](../ipc/envelope.md) is drawn from the secure RNG, so
  entropy comes up first, both inside `init_core_services`. Both are fatal on failure.
- **The BSP before the scheduler.** The scheduler and everything per-CPU need the
  [bootstrap processor's per-CPU data](../smp/per-cpu.md) established first, which is why `init_bsp`
  precedes `sched::init` in `init_core_services`.
- **The virtual-memory manager before any process.** `init_vm_and_protection` runs before
  `init_platform_baseline` and `init_process_runtime`; no process, not even init, can be created before
  the [paging manager](../memory/unified-vm.md) and the kernel address space exist.
- **The framebuffer MMIO map after paging.** The framebuffer is mapped as MMIO, which needs the paging
  machinery, so the early step (1) only records it and the real mapping happens at step 7, after
  `init_vm_and_protection`. Doing it earlier failed on real GOP framebuffers because the page tables were
  not ready.
- **IO-APIC routing after unified paging.** `init_device_routing` (step 8) programs the IO-APIC, an MMIO
  write, so it must run after the MMIO window is mappable. This is why device IRQ binds depend on it
  running at the right point in the order.
- **The ELF loader and kernel keys before spawning.** The [capsule loader](../elf-loader/README.md) and
  the [signing key](../crypto/asymmetric.md) come up in `init_process_runtime`, and no capsule is spawned
  until `microkernel_main` runs afterward.

## Fatal failures

The security-critical steps halt rather than degrade: a missing precondition writes its stage to the boot
log and serial and stops. These are the invariants the kernel refuses to run without: entropy, a keyed
IPC path, a live bootstrap processor, a working scheduler, and a working virtual-memory manager. There is
no partial-boot mode past them. The same discipline appears in `microkernel_main`, where a failure to
create the init process, its address space, or its kernel stack halts.

## Crossing into userspace

After step 10 the core is ready and control passes to `microkernel_main`
(`src/kernel_core/init/entry/microkernel_main.rs:22`). It spins a short settling window (~2.5 s,
`:28`), creates the first process, `init`, with `create_process("init", Running, High)` (`:38`), gives it
a fresh address space (`:48`) and a kernel stack (`:53`), makes it current (`:58`), and calls
`crate::userspace::run_init()` (`:61`). The [userspace init](userspace-init.md) page picks up from there:
`run_init` (`src/userspace/init/entry.rs:20`) spawns the capsule fleet, the first capsule being `ramfs`
(`entry.rs:26`, `spawn_plan::spawn_ramfs`). The initial capsule set is not read from an external
manifest; it is the statically compiled, feature-gated spawn plan, and each capsule's embedded ELF is
verified against its manifest and attestation trailer at spawn.

## Kernel measurement and the TPM

The kernel does not measure itself into the TPM during init. On x86_64 the kernel image was already
measured (BLAKE3), dual-signature-verified, STARK-attested, and rollback-checked by the
[bootloader](bootloader.md) before the jump; `microkernel_init` performs no PCR extend. The kernel only
*reads* the recorded results from the handoff (`src/entry/security.rs:19`), logged once between
`init_core_systems` and `microkernel_init` (`log_security_status`, `src/nonos_main.rs:92`). The kernel
reaches the TPM later and on demand, through the `sys_attest_doc` syscall
(`src/syscall/microkernel/attest_doc.rs:35`), not as part of the boot order. Capsule attestation, by
contrast, is verified at spawn, one capsule at a time, in the [userspace init](userspace-init.md) path.

## Debugging kernel init

On a machine with no serial port the kernel's on-screen text log is deliberately off: `init_after_fb`
(`src/sys/boot_log/init.rs`) leaves the bootloader's verified-boot splash in the framebuffer for the
compositor to build on, so the kernel log is a serial-only stream. Each successful stage prints a
bracketed tag line through `boot_log::ok`. A fatal stop is a `[ERROR]` line followed by a `[FATAL]
<stage>: <detail>` on serial, so the last two serial lines name exactly which precondition failed. The
common early symptom, a black screen with the bootloader splash frozen and the kernel serial silent after
the handoff, means bring-up died before or at the first `boot_log` call rather than at any specific stage.

## Source map

```
  src/nonos_main.rs                                   kernel_entry (x86), init_core_systems, boot_microkernel
  src/boot/main/core_init/init_core_systems.rs        the x86 pre-init (GDT, IDT, APIC, ACPI, sti)
  src/arch/aarch64/boot/entry.rs                       the aarch64 entry that converges on microkernel_init
  src/kernel_core/init/entry/microkernel_init.rs       the ordered core bring-up
  src/kernel_core/init/entry/init_core_services.rs     RNG, IPC secret, SMP BSP, scheduler, clock (the fatal steps)
  src/kernel_core/init/entry/init_vm_and_protection.rs unified VM / paging and protection
  src/kernel_core/init/entry/init_runtime.rs           device routing and process runtime (ELF loader, kernel keys)
  src/kernel_core/init/entry/microkernel_main.rs       create init, enter userspace
  src/kernel_core/init/platform/baseline.rs            broker, entropy, boot nonce, token signing key
```

Every reference above is verified against those trees. The handoff and its verification that precede this
sequence are on the [bootloader](bootloader.md) and [handoff](handoff.md) pages; the userspace transition
that follows it is on the [userspace init](userspace-init.md) page.

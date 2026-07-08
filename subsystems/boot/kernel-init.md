# Kernel Initialization

When the bootloader hands control to the kernel, `microkernel_init` brings the system up in a fixed
order, each step a precondition for the next, and any step that cannot complete is fatal. This page
walks that sequence. The code is `src/kernel_core/init/entry.rs`.

## The sequence

`microkernel_init(handoff)` (`entry.rs:24`) runs the bring-up in order:

```
  1.  init_arch_memory_and_framebuffer   EFI memory map -> frame allocator; early framebuffer
  2.  boot_log init                       the on-screen and serial boot log
  3.  init_arch_firmware                  ACPI / SMBIOS tables
  4.  hostname_init
  5.  crypto::rng::init_rng               FATAL if entropy is unavailable
  6.  ipc::init_ipc_secret                FATAL: seed the IPC MAC key
  7.  smp::init_bsp                       FATAL: bring up the bootstrap processor
  8.  sched::init                         the scheduler
  9.  clock::init                          TSC frequency + boot epoch from the handoff
  10. memory::init_unified_vm             FATAL: the paging manager and kernel address space
  11. init_arch_framebuffer               MMIO-map the framebuffer (needs paging)
  12. init_broker_irq_routing            the IO-APIC routing for device IRQs
  13. process::init_process_management    the process tables
  14. elf::loader::init_elf_loader        the capsule loader
  15. crypto::kernel_keys::init           the kernel Ed25519 signing keypair
  16. start_secondary_cpus                the application processors
```

Each numbered step depends on the ones before it, which is why the order is fixed and not merely
conventional.

## Why the order

The dependencies are load-bearing, and the code comments call the sharp ones out:

- **RNG before the IPC secret.** The [IPC MAC key](../ipc/envelope.md) is drawn from the secure RNG,
  so entropy must be up first; both are fatal on failure, because the kernel will not run with an
  unseeded generator or an unkeyed IPC path.
- **The BSP before the scheduler.** The scheduler and everything per-CPU need the
  [bootstrap processor's per-CPU data](../smp/per-cpu.md) established first.
- **The virtual-memory manager before any process.** `init_unified_vm` brings up the
  [paging manager](../memory/unified-vm.md) and the kernel's own address space; no process, not even
  init, can be created before it, so process-management init and the userspace init process both come
  after.
- **The framebuffer MMIO map after paging.** The framebuffer is mapped as MMIO, which needs the
  paging machinery, so the early step only records it and the real mapping happens at step 11; doing
  it earlier failed on real GOP framebuffers because the page tables were not ready.
- **The ELF loader before the kernel keys are used, and both before spawning.** The
  [capsule loader](../elf-loader/README.md) and the [signing key](../crypto/asymmetric.md) must exist
  before any capsule is verified and loaded.

## Fatal failures

Steps marked fatal call `fatal` (`entry.rs:73`), which writes the stage and detail to the boot log
and serial and halts. These are the invariants the kernel refuses to run without: entropy, a keyed
IPC path, a live bootstrap processor, and a working virtual-memory manager. The kernel does not limp
along in a degraded state past any of them; it stops with a legible reason. After step 16 the core is
ready, and control passes to [userspace init](userspace-init.md).

## Source

```
  src/kernel_core/init/entry.rs        microkernel_init, the ordered sequence, fatal
  src/kernel_core/init/memory.rs        the early arch memory bring-up
  src/kernel_core/init/framebuffer.rs   the framebuffer MMIO map
  src/kernel_core/init/start_secondary.rs   the AP bring-up
```

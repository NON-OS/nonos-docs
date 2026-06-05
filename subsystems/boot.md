# Boot and Init

This page traces control from the kernel entry point to the first capsule, step
by step. The [architecture overview](../architecture/overview.md) gives the shape
in section 4; here is every stage with its function and source. The path is a
fixed sequence in three phases: core CPU bring-up, microkernel services, then the
init process and the drop to user mode.

The sequence below is the x86_64 path. ACPI discovery and the IO-APIC are x86_64
specifics; aarch64 and riscv64 reach the same milestones through a flattened
device tree and their own interrupt controllers.

---

## Phase 1: core systems

Entry is `kernel_entry` (`src/nonos_main.rs:39`), which immediately calls
`init_core_systems` (`src/boot/main/core_init.rs:21`). That function brings the
CPU from the bootloader handoff to a state where interrupts, the heap, and the
syscall boundary all work.

```
  init_core_systems                       src/boot/main/core_init.rs:21
    serial::init                          COM1 up, earliest possible output
    time::timer::init_boot_time           TSC boot timestamp
    tsc::init_default
    gdt::init                             segments and the TSS
    syscall::init                         STAR / LSTAR / CSTAR MSRs        :31
    idt::setup                            early IDT
    heap::manager::init_bootstrap         the global allocator              :42
    interrupts::init_idt                  full IDT with all handlers        :44
    init_acpi_tables                      RSDP from the handoff
    apic::init                            local APIC
    preemption::install_on_bsp            100 Hz timer, IRQ-0 -> tick       :49
    sti                                   interrupts enabled                :58
    init_memory_encryption
    pci::init                             enumerate the PCI bus
    seed_hardware_broker                  the broker's device view
    init_entropy
    init_boot_session_nonce               binds capability tokens to this boot
    init_token_signing_key                the key behind every token MAC
```

The order is not arbitrary. The heap is bootstrapped before the full IDT so
handlers can allocate. The syscall MSRs are programmed before any capsule could
exist. The boot session nonce and token signing key are established here so that
every capability token minted later is bound to this boot and unforgeable across
reboots (see [capabilities and tokens](../security/capabilities-and-tokens.md)).

## Phase 2: microkernel services

`microkernel_init` (`src/kernel_core/init/entry.rs:26`) builds the services the
kernel runs on: memory, the scheduler, the clock, unified paging, interrupt
routing, the process table, and the trust keys.

```
  microkernel_init(handoff)               src/kernel_core/init/entry.rs:26
    init_arch_memory_and_framebuffer      physical memory from the EFI map
    init_framebuffer
    boot_log::init_after_fb               on-screen boot log
    rng::init_rng
    nonos_channel::init_ipc_secret
    smp::init_bsp                         the bootstrap processor
    sched::init                           the scheduler                     :43
    clock::init
    unified::init_unified_vm              upper-half paging, low-half torn down  :50
    ioapic::init_from_acpi                interrupt routing table            :53
    init_process_management               the process table
    elf::loader::init_elf_loader
    kernel_keys::init                     the capsule trust keys
    start_secondary::start_secondary_cpus the AP cores
```

Two ordering facts matter. `init_unified_vm` runs before
`ioapic::init_from_acpi`, because programming an IO-APIC redirection entry is an
MMIO write and the MMIO window only becomes mappable after unified paging is
established (`entry.rs:50` then `:53`). Initialise the IO-APIC earlier and the
write page faults. And the scheduler is initialised before paging is finalised,
but it does no useful work until the runqueue is populated in phase 3.

See [memory](memory.md) for what `init_unified_vm` does and
[interrupts](interrupts.md) for `init_from_acpi`.

## Phase 3: the init process and user mode

`microkernel_main` (`src/kernel_core/init/entry.rs:112`) creates the first
process and hands it the CPU.

```
  microkernel_main                        src/kernel_core/init/entry.rs:112
    create_process("init", Running, High) reserved pid 1
    create_address_space(init_pid)        page tables for init
    allocate_kernel_stack(init_pid)
    CURRENT_PID.store(init_pid)
    userspace::run_init                    enter the supervisor
```

`run_init` (`src/userspace/init/entry.rs:20`) is the supervisor. It spawns the
system capsules through their spawn plans, then enters `init_loop`, the
supervisor loop that services IPC for the lifetime of the system. Each capsule it
spawns goes through the verified-spawn gate
([capsules and trust](../security/capsules-and-trust.md)): the capsule is created
`Ready`, added to the runqueue, and only executes when the scheduler reaches it
and performs the first-entry context switch that drops to ring 3.

---

## The drop to ring 3

A freshly spawned capsule is not running yet. Install left an `iretq` frame in
its process control block (the entry RIP and user RSP). The first time the
scheduler selects it, the context switch dispatcher
(`src/arch/x86_64/context/switch/dispatch.rs:39`) loads that frame and executes
`iretq`, which pops RIP, CS, RFLAGS, RSP, and SS off the kernel stack and
transfers to the capsule's ELF entry point in user mode with interrupts enabled.
The initial frame is built by `setup_initial_user_pcb`
(`src/arch/x86_64/context/setup.rs:36`), which rejects any entry or stack pointer
above the canonical user boundary before it constructs the frame, so a capsule
cannot be set up to begin executing in kernel space.

From that point the capsule runs its own `main`, makes syscalls across the
boundary documented in the [ABI reference](../abi/syscalls.md), and is scheduled
alongside every other capsule by the [scheduler](scheduler.md).

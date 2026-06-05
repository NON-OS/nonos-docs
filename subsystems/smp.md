# SMP and Per-CPU

NØNOS runs on multiple cores. The bootstrap processor brings itself up, discovers
the other cores from ACPI, and starts them through the standard INIT-SIPI-SIPI
sequence. Each core has its own per-CPU data, its own LAPIC timer, and its own
scheduler queue. This page covers detection, AP bring-up, per-CPU storage, and
the per-CPU scheduler.

This is the x86_64 path. The arch boundary's `current_cpu_id` primitive
([overview](../architecture/overview.md)) is the portable face of the per-CPU
identity described here.

---

## Detection

CPU detection prefers the authoritative source, the ACPI MADT, over CPUID
topology leaves (`src/smp/topology/detection.rs:29`). It reads the enabled
processors from `acpi::processors()`, builds the list of application processors
(every enabled APIC id except the BSP), and caps the count at `MAX_CPUS = 256`
(`src/smp/constants.rs:17`). CPUID leaves 0xB, 0x04, and 0x01 are a fallback when
ACPI topology is thin, and are also used to detect hyperthreading.

## Bootstrap processor

`init_bsp` (`src/smp/init/bsp.rs:24`) runs on the boot core during microkernel
init:

```
  init_bsp()
    detect_cpus()                     count cores from the MADT
    BSP_APIC_ID = this core's APIC id
    initialise the cpu 0 descriptor
    percpu::init_bsp()                set up this core's per-CPU data
```

After this the BSP has its per-CPU data installed and knows how many cores to
start.

## Starting the application processors

`start_aps` (`src/smp/init/ap_start.rs:22`) walks the AP list and starts each one
through `ap_unit::start` (`src/smp/init/ap_unit.rs:27`):

```
  ap_unit::start(cpu_id, apic_id, boot)
    allocate this AP's per-CPU stack
    write the AP boot context: PML4, stack top, entry pointer, cpu_id
    apic::start_ap(apic_id, start_page)        the INIT-SIPI-SIPI sequence
    wait for the AP to report CpuState::Online (bounded by a TSC timeout)
```

The wake sequence is the architectural one (`src/arch/x86_64/interrupt/apic/ipi_ap.rs:23`):

```
  INIT assert            ICR delivery INIT, level assert
  INIT deassert          after a 10 us delay
  SIPI                   vector = trampoline page (0x8000), then 200 us
  SIPI                   a second SIPI, the standard double send
```

The AP starts executing 16-bit code at the trampoline page
(`AP_TRAMPOLINE_ADDR = 0x8000`, `src/smp/constants.rs:21`), brings itself to long
mode using the PML4 the BSP wrote into its boot context, and jumps to `ap_entry`.

## AP entry

`ap_entry` (`src/smp/ap.rs:24`) is where a freshly started core joins the system:

```
  ap_entry()
    initialise the local APIC
    read this core's APIC id
    cpu::init_ap()                    GDT and TSS for this core
    idt::load_on_ap()                 the shared IDT
    percpu::init_ap(cpu_id)           this core's per-CPU data
    sched::init_ap_scheduler(cpu_id)  this core's run queue
    install_on_ap()                   this core's LAPIC timer at 100 Hz
    set this core's state to Online
    increment AP_STARTUP_BARRIER
    enter the idle loop
```

Each AP installs its own LAPIC timer, so every core is preempted on its own 10 ms
cadence. The startup barrier lets the BSP confirm a core came up before relying
on it. A core that does not reach `Online` within the TSC timeout is treated as
failed rather than hanging the boot.

---

## Per-CPU data

Each core has a 4 KiB-aligned `PerCpuData` (`src/smp/percpu/types.rs:28`),
reached through the GS base register:

```
  PerCpuData
    self_ptr           points at this struct, so GS-relative loads resolve it
    cpu_id, apic_id
    current process and thread pointers
    kernel_stack_top
    irq nesting depth
    scheduler lock
    random_state       a per-CPU xorshift seed
```

The descriptors for every core live in one global array, `CPU_DESCRIPTORS[MAX_CPUS]`
(`src/smp/state.rs:21`), alongside `CPU_COUNT` and `CPUS_ONLINE`.

### The GS base and swapgs

The GS base points at the current core's `PerCpuData` while in the kernel, and is
zero in user mode (`src/smp/percpu/operations.rs:29`):

```
  kernel mode   GS_BASE = &PerCpuData[cpu_id]   KERNEL_GS_BASE = 0
  user mode     GS_BASE = 0                     KERNEL_GS_BASE = &PerCpuData
```

The kernel executes `swapgs` on every transition between user and kernel, which
swaps the active and shadow GS bases. So a syscall or interrupt entry runs one
`swapgs` to make per-CPU data reachable, and the matching return runs another to
restore the user GS. This is how a handler finds the current core's state with a
single GS-relative load, with no lookup, even though many cores run the same code.

`current_cpu_id` resolves the core by reading the APIC id out of CPUID and mapping
it to a dense cpu id (`src/arch/x86_64/abi.rs:59`); `percpu::current` returns the
`PerCpuData` for that id.

---

## Per-CPU scheduling

The scheduler is per-CPU with load balancing
(`src/process/scheduler/smp`). Each core has its own run queue, set up by
`init_ap_scheduler` (`src/process/scheduler/smp/api.rs:29`), and runs work from
its local queue. When a core's queue runs dry it pulls work from a busier core
through `try_load_balance`, so a core does not sit idle while another is backed
up. The single-core selection logic, the five priority classes and the
round-robin within a class, is described on the [scheduler page](scheduler.md);
on a multiprocessor machine that selection runs per core over the local queue.

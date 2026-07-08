# SMP: Per-CPU Data and CPU Identity

On a multicore machine each CPU has its own private per-CPU structure, and each CPU can
identify which core it is through the architecture boundary. This page documents the
per-CPU data, how a CPU learns its own identity, and the per-CPU address-space id that
scopes TLB shootdowns. The code is under `src/smp/`. The kernel supports up to
`MAX_CPUS = 256` (`src/smp/constants.rs:17`).

## The per-CPU structure

Each CPU has a `PerCpuData`, page-aligned so it sits on its own cache lines and page
(`src/smp/percpu/types.rs:29`):

```
  #[repr(C, align(4096))]
  PerCpuData
    self_ptr                u64        pointer to this structure
    cpu_id / apic_id        u32        the dense index and the hardware APIC id
    current_process         AtomicU64  the process running on this CPU
    current_thread          AtomicU64
    kernel_stack_top        u64        this CPU's kernel stack
    user_stack_saved        u64
    syscall_scratch         [u64; 4]   scratch for the syscall trampoline
    irq_nesting             u32        interrupt nesting depth
    sched_lock_held         u32
    random_state            AtomicU64  per-CPU RNG state
    last_tick_tsc           AtomicU64
    interrupt_disable_depth u32
    active_asid             AtomicU32  the address space executing here
    _reserved               padding to a full 4096-byte page
```

The structure holds exactly the state that must be private to a CPU: which process and
thread it is running, its kernel stack, its syscall scratch, its interrupt nesting and
disable depth, and its own RNG state. Because it is per-CPU, these are read and written
without locking or atomics for the fields only ever touched by the owning CPU, and with
atomics for the ones another CPU may read, such as `current_process` and `active_asid`.
The page alignment keeps one CPU's structure off another's cache lines.

## CPU identity

A CPU learns which core it is through the arch boundary (`src/smp/cpu.rs:22`):

```
  cpu_id():
      id = arch::cpu::get_cpu_id()          APIC id, MPIDR_EL1, or hart id
      apic_to_cpu_id(id).unwrap_or(0)        map the hardware id to a dense index
```

`get_cpu_id` is the arch-neutral call: on x86_64 it reads the Local APIC id, on aarch64
`MPIDR_EL1`, on riscv64 the hart id. Those hardware ids are not necessarily dense or
zero-based, so `apic_to_cpu_id` maps the hardware id to a dense `0..CPU_COUNT` index by
scanning the CPU descriptors, and that dense index is what the rest of the kernel uses,
for example to index the per-CPU current-pid array in the [process table](../process/process-table.md).
`is_bsp` (`cpu.rs:53`) reports whether the calling CPU is the bootstrap processor by
comparing its hardware id against the recorded `BSP_APIC_ID`.

## The active address space

The `active_asid` field is the one other CPUs read, and it exists for TLB coherency
(`percpu/types.rs:19`). It records the address-space id currently executing on this CPU,
updated by `paging::manager::switch_address_space` on every context switch into a
process. The reserved value `ASID_NONE` (zero) means no user CR3 is active on this CPU,
which is the state at boot before any process runs and after a CPU has driven a process
off without yet loading another. The [TLB shootdown](tlb-shootdown.md) broadcaster reads
these to decide which CPUs a per-address-space invalidation needs to reach: a user-VA
flush does not need to reach a CPU whose `active_asid` is `ASID_NONE` or a different
address space, while kernel-VA flushes are not ASID-keyed and reach every online CPU.

## Source

```
  src/smp/percpu/types.rs       PerCpuData and ASID_NONE
  src/smp/percpu/operations.rs  current() and the per-CPU accessors
  src/smp/cpu.rs                cpu_id, apic_to_cpu_id, is_bsp
  src/smp/constants.rs          MAX_CPUS
```

# SMP: Per-CPU Data and CPU Identity

On a multicore machine each CPU has its own private per-CPU structure, and each CPU can
identify which core it is through the architecture boundary. This page documents the
per-CPU data, how a CPU learns its own identity, and the per-CPU address-space id that
scopes TLB shootdowns. The code is under `src/smp/`. The kernel supports up to
`MAX_CPUS = 256` (`src/smp/constants.rs:17`).

## The per-CPU structure

Each CPU has a `PerCpuData`, page-aligned so it sits on its own cache lines and page
(`src/smp/percpu/types.rs:28`):

```
  #[repr(C, align(4096))]
  PerCpuData
    self_ptr                u64        pointer to this structure
    cpu_id                  u32        the dense index
    apic_id                 u32        the hardware APIC id
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
    time_slice              AtomicU64  ticks left in the running task's slice
    need_resched            AtomicU32  reschedule at the next safe point
    tlb_flush_pending       AtomicU32  this CPU is a target of a live shootdown
    _reserved               [u8; 4096 - 132]   padding to a full page
```

The structure holds exactly the state that must be private to a CPU. Three of these fields
are load-bearing for correctness under more than one CPU and were once mistakenly single
globals (comment `types.rs:48`): `time_slice` and `need_resched` are the scheduler's
per-CPU slice counter and reschedule flag, once shared so that every CPU's tick decremented
one counter (N cores burned a slice N times too fast) and a reschedule raised anywhere was
seen everywhere; `tlb_flush_pending` (comment `types.rs:56`) is the per-CPU target flag the
[TLB shootdown](tlb-shootdown.md) originator sets before publishing a request, which is what
makes each flush correctly targeted and safe to run twice. The three were appended after the
offsets `layout.rs` asserts, so the assembly that addresses this block is unaffected. Fields
only ever touched by the owning CPU (kernel stack, syscall scratch, nesting, disable depth,
RNG state) are read and written without atomics; the ones another CPU may read
(`current_process`, `active_asid`, `time_slice`, `need_resched`, `tlb_flush_pending`) are
atomics. The page alignment keeps one CPU's structure off another's cache lines.

## CPU identity

A CPU learns which core it is through an interrupt-controller-id scan, not a per-CPU
register read (`src/smp/cpu_id.rs:38`):

```
  cpu_id():                                      # cpu_id.rs:38
      apic_id = arch::cpu::get_cpu_id()          APIC id, MPIDR_EL1, or hart id
      if apic_to_cpu_id(apic_id) is Some(index): return index   # scan the descriptors
      if CPU_COUNT == 0: return 0                boot CPU, before init_bsp filled its descriptor
      unregistered(apic_id)                       # print FATAL and halt; never guess
```

`get_cpu_id` is the arch-neutral call: on x86_64 it reads the Local APIC id, on aarch64
`MPIDR_EL1`, on riscv64 the hart id. Those hardware ids are not necessarily dense or
zero-based, so `apic_to_cpu_id` (`cpu.rs:25`) maps the hardware id to a dense `0..CPU_COUNT`
index by scanning the CPU descriptors, and that dense index is what the rest of the kernel
uses, for example to index the per-CPU current-pid array in the
[process table](../process/process-table.md).

Two design points here are deliberate and worth stating. First, the derivation is an APIC
scan rather than the one-instruction `gs:`-relative read precisely because a `gs:` read is
only correct while kernel code always runs on the kernel GS base, and this tree does not yet
enforce that: NMI, `#NM`, `#MF`, `#VE`, and the keyboard, mouse, and int-0x80 vectors are
reachable from CPL=3 with no swapgs trampoline, so a `gs:` read there would fault or read the
user base (comment `cpu_id.rs:24`). Second, the answer is never guessed: the code used to end
in `unwrap_or(0)`, but 0 is the one answer that must never be assumed, because the current
process is tracked per CPU and every capability check in the syscall layer is keyed on it, so
a CPU quietly claiming to be the boot CPU would read and write another CPU's current process
and be granted its authority (comment `cpu_id.rs:32`). An unregistered CPU therefore halts in
`unregistered` (`cpu_id.rs:54`) rather than run as core 0.

`is_bsp` (`cpu.rs:43`) reports whether the calling CPU is the bootstrap processor by
comparing its hardware id against the recorded `BSP_APIC_ID`, not against 0.

## The active address space

The `active_asid` field (`percpu/types.rs:47`) is one of the fields other CPUs read, and it
exists for TLB coherency. It records the address-space id currently executing on this CPU,
updated by `paging::manager::switch_address_space` on every context switch into a
process. The reserved value `ASID_NONE` (zero) means no user CR3 is active on this CPU,
which is the state at boot before any process runs and after a CPU has driven a process
off without yet loading another (`ASID_NONE`, `percpu/types.rs:26`). The [TLB shootdown](tlb-shootdown.md) broadcaster reads
these to decide which CPUs a per-address-space invalidation needs to reach: a user-VA
flush does not need to reach a CPU whose `active_asid` is `ASID_NONE` or a different
address space, while kernel-VA flushes are not ASID-keyed and reach every online CPU.

## Security analysis

Per-CPU state is a correctness and isolation mechanism rather than a privilege boundary: none of it is
reachable from ring 3. Its properties are about keeping one CPU's state from corrupting another's and
about a CPU knowing which core it actually is. Three hold.

**Per-CPU isolation is by page-aligned ownership, not by locking.** `PerCpuData` is `#[repr(C,
align(4096))]` (`src/smp/percpu/types.rs:28`), so each CPU's structure sits on its own cache lines and its
own page. The fields only ever touched by the owning CPU (kernel stack, syscall scratch, interrupt
nesting and disable depth, RNG state) are read and written without locking or atomics, which is sound
precisely because no other CPU touches them. The handful another CPU may read (`current_process`,
`active_asid`) are atomics. The isolation is structural: a CPU indexes its own record and does not reach
into another's, so there is no shared mutable field to race on for the non-atomic ones.

**A CPU's identity is derived from hardware, then made dense, and never guessed.** `cpu_id`
(`src/smp/cpu_id.rs:38`) reads the hardware id through the arch facade (`get_cpu_id`, the Local APIC id on
x86_64) and maps it to a dense `0..CPU_COUNT` index with `apic_to_cpu_id` by scanning the CPU
descriptors. This matters because the boot CPU's APIC id is not guaranteed to be 0: firmware assigns APIC
ids, and the dense index the rest of the kernel uses to index per-CPU arrays is a separate space from the
hardware id. The old `unwrap_or(0)` was removed for a concrete reason (comment `cpu_id.rs:32`): 0 is the
one index that must never be assumed, because the per-CPU current process and every syscall capability
check are keyed on it, so a CPU quietly resolving to 0 would read and write another CPU's process and
inherit its authority. An unregistered CPU halts in `unregistered` (`cpu_id.rs:54`) instead. `is_bsp`
(`cpu.rs:43`) reflects the same honesty by comparing `get_cpu_id()` against the recorded `BSP_APIC_ID`
(stored during `init_bsp`, `src/smp/init/bsp.rs:31`) rather than against 0.

**`active_asid` is the only field published for cross-CPU reads, and it is an atomic with a defined
idle value.** It records the address space executing on this CPU, updated by
`switch_address_space` on every context switch, and `ASID_NONE` (zero) means no user CR3 is active
(`percpu/types.rs:19`). The [TLB shootdown](tlb-shootdown.md) filter reads it to decide which CPUs a
per-address-space invalidation must reach. The honest boundary here is which accessor an entry path uses.
Identity itself (`cpu_id`) is safe on any path because it resolves by an APIC-id scan, not a `gs:` read,
which is exactly why it is a scan (`cpu_id.rs:24`). A `self_ptr`/`gs:`-based struct accessor, by contrast,
would depend on the kernel GS base being loaded first, so on the entry paths that lack a swapgs trampoline
(NMI, `#NM`, `#MF`, `#VE`, keyboard, mouse, int-0x80) such an accessor would index the wrong CPU; those
paths must reach per-CPU state through the scan-based index, not a raw `gs:` load.

## Debugging per-CPU state

Per-CPU bugs rarely print; they show as one core behaving differently from the others, so they are
diagnosed by asymmetry. The startup path does log identity, which is the anchor for everything else:
`init_bsp` prints `[SMP] BSP initialized: APIC ID=<n>, <k> CPUs detected` (`src/smp/init/bsp.rs`), and AP
bring-up prints `[SMP] AP <cpu_id> online (APIC <n>)` (`src/smp/init/ap_unit.rs`). Read together these
tell you the mapping from dense `cpu_id` to hardware APIC id that `apic_to_cpu_id` built, which is the
first thing to check when a per-CPU array looks like it is indexing the wrong core.

The characteristic failure modes:

- **A wrong or duplicated `cpu_id`.** If two CPUs resolve to the same dense index, `apic_to_cpu_id` found
  two descriptors with the same APIC id or the descriptor table was misfilled during bring-up. The
  symptom is two cores sharing one per-CPU record, which corrupts stacks and scheduler state. The
  `[SMP] AP ... online (APIC ...)` lines are where you confirm each APIC id is distinct.
- **`is_bsp` disagreeing with expectation.** On hardware where the BSP is not APIC id 0, code that assumed
  the BSP is core 0 will act on the wrong CPU while `is_bsp` (which compares against the recorded
  `BSP_APIC_ID`) stays correct. A divergence between the two is the tell that some other code hard-coded
  0 instead of asking `is_bsp`.
- **A per-CPU read returning another core's data.** Because `cpu_id` is an APIC scan, the identity index
  itself is not the culprit; this presents when a `gs:`/`self_ptr`-based struct accessor is used on an
  entry path that has not loaded the kernel GS base, giving one core's counters or `active_asid` while the
  others are fine. The fix is to reach per-CPU state through the scan-based index on those paths, not a raw
  `gs:` load.

## Source map

```
  src/smp/percpu/types.rs       PerCpuData, ASID_NONE, time_slice/need_resched/tlb_flush_pending
  src/smp/percpu/operations.rs  current() and the per-CPU accessors
  src/smp/cpu_id.rs             cpu_id (APIC scan, fatal-on-unregistered)
  src/smp/cpu.rs                apic_to_cpu_id, is_bsp, current_cpu
  src/smp/init/bsp.rs           init_bsp, BSP_APIC_ID recorded from apic::id()
  src/smp/init/ap_unit.rs       AP descriptor configuration and the online log
  src/smp/constants.rs          MAX_CPUS
```

Every reference above is verified against those trees. The AP bring-up that fills these descriptors is on
the [TLB shootdown](tlb-shootdown.md) page's neighbours in this section, the `active_asid` field is
consumed by the [TLB shootdown](tlb-shootdown.md) filter, and the dense `cpu_id` indexes the per-CPU
current-pid array in the [process table](../process/process-table.md).

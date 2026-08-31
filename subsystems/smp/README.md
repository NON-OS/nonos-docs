# SMP

How NØNOS runs on more than one CPU: the per-CPU data each core keeps private, how a
core identifies itself through the architecture boundary, and the cross-CPU TLB
invalidation that keeps address spaces coherent when a mapping changes.

## Current status: single-core on x86_64

State the honest position first. On the shipping x86_64 build the system runs on one
core. The boot handoff hardcodes `CpuTopology { boot_cpu_id: 0, cpu_count: 1 }`
(`src/boot/handoff/kernel_handoff/x86_64/builders.rs:44`), so `cpus_online()` is 1 and
every cross-CPU path below short-circuits to its local-only branch. `start_secondary_cpus`
exists (`src/kernel_core/init/start_secondary.rs`) and the AP bring-up, trampoline, and IPI
machinery are all present and compiled, but secondary-core bring-up is **not proven working
on x86_64**: nothing in the shipping path starts an application processor. The structures
documented here (per-CPU data, CPU-identity derivation, TLB shootdown) are implemented and
correct as code, and they are what a working multi-core boot would use, but the multi-core
path itself is dormant. Read every "each CPU" and "other CPUs" statement below as describing
the design, not a behaviour the current build exercises.

| Page | What it covers |
|------|----------------|
| [per-cpu.md](per-cpu.md) | The page-aligned `PerCpuData`, CPU identity through the arch boundary (APIC id, MPIDR, hart id) mapped to a dense index, and the per-CPU `active_asid`, `time_slice`, and `need_resched` fields. |
| [tlb-shootdown.md](tlb-shootdown.md) | The IPI broadcast-and-wait that invalidates a mapping across every CPU, the acknowledging IPI handler, the `invalidate_page`/`invalidate_all` primitives, the fail-hard timeout, and how the address-space scope is decided. |

Multicore bring-up, starting the application processors from the bootstrap processor,
lives under `src/smp/init/` (the BSP and AP init), `src/smp/ap/`, and `src/smp/trampoline/`
(the real-mode trampoline the APs start from), and the inter-processor interrupt machinery
the shootdown uses is under `src/smp/ipi_dispatch/` and `src/smp/ipi_handler.rs`. The
per-CPU current process this section's `active_asid` complements is on the
[process table](../process/process-table.md) page.

## Sources

The code for this subsystem lives under `src/smp/`: `percpu/` (the per-CPU data), `cpu_id.rs`
and `cpu.rs` (identity), `ipi_dispatch/` and `ipi_handler.rs` (inter-processor interrupts),
`init/` and `ap/` (bring-up), and `trampoline/` (the AP start trampoline). The TLB shootdown
itself is not under `src/smp/`; it lives with the paging manager at
`src/memory/paging/manager/shootdown.rs` and uses the `src/smp/` IPI transport. Every page is
verified against those trees with `file:line` references.

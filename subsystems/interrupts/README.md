# Interrupts

How the CPU enters the kernel on a trap, and how the kernel gets back out. Every exception,
IRQ, and syscall gate is one entry in the interrupt descriptor table; the user-reachable
entries run through naked assembly trampolines that switch the per-CPU base and, for the timer,
snapshot the preempted capsule; the Rust handlers decide recovery; and two controllers deliver
and acknowledge the lines.

| Page | What it covers |
|------|----------------|
| [idt.md](idt.md) | The vector map, `build_idt`, gate and IST assignment, the ring-3 syscall gate, and the naked-trampoline vs `x86-interrupt`-wrapper split. |
| [trampolines.md](trampolines.md) | The `swapgs`-on-CPL3 pattern, the `fxsave` SIMD preservation, and the timer trampoline that captures the preempted capsule's `UserContext` for preemption. |
| [handlers.md](handlers.md) | The page-fault demand/guard/terminate-vs-panic path, the double-fault halt, the IRQ handlers, and the shared interrupt-context and end-of-interrupt. |
| [controllers.md](controllers.md) | The 8259 PIC remap to vectors 32-47, the local APIC façade, and the gate that acknowledges an interrupt to exactly one live controller. |
| [allocation.md](allocation.md) | The runtime vector pool, the reserved-and-handler registry, the fixed reservations, and `allocate_vector` / `free_vector`. |
| [safety.md](safety.md) | The RAII interrupt guard that restores prior state, and the per-CPU interrupt-context nesting depth read through `gs:8`. |

The property that runs through the section is that entry from ring 3 is never trusted to leave
the CPU in a safe state: the trampoline switches the GS base before any handler reads per-CPU
memory, the fatal exceptions run on dedicated stacks so a fault in a fragile window lands on
known-good memory, and the timer path captures enough state to resume a capsule exactly where
it was preempted. That last piece is the hinge of preemptive multitasking and is picked up by
the [scheduler](../scheduler/preemption.md) and the [context switch](../process/context-switch.md).

## Sources

The code for this subsystem lives under `src/interrupts/`: `idt/` (the table and vector map),
`isr/` (the naked trampolines and wrappers), `handlers/` (the exception and IRQ bodies),
`pic/` and `apic/` (the controllers), `allocation/` (the vector pool), `safety/` (the guard
and interrupt context), and `stats/` (the counters). The IST slots come from the
[GDT](../smp/README.md), and the broker IRQ entries from `src/arch/x86_64/interrupt/broker`.
Every page is verified against those trees with `file:line` references.

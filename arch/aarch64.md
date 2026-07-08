# The aarch64 Backend

aarch64 is an architecture-ready backend: it implements the full `ArchOps` boundary and carries the
ARM-specific machinery (the GIC, PSCI, the MMU, the generic timer), so the kernel compiles and links for
64-bit ARM. It is the next target after x86_64 in the multi-architecture plan, ahead of runtime bring-up
on QEMU and then hardware. This page documents it honestly. The code is under `src/arch/aarch64/`.

## The ArchOps implementation

The `Aarch64` backend (`src/arch/aarch64/abi/archops.rs:24`) implements the same eight primitives as
x86_64, delegating each to a dedicated backend module:

```
  halt()                -> halt::halt()
  enable_interrupts()   -> irq_enable::enable()        (unmask via DAIF)
  disable_interrupts()  -> irq_disable::disable()
  interrupts_enabled()  -> irq_state::enabled()
  current_cpu_id()      -> cpu_id::current()           (MPIDR affinity)
  read_time_counter()   -> time::counter()             (the generic timer)
  flush_tlb_one(addr)   -> tlb::flush_one(addr)
  switch_address_space  -> address_space::switch(root) (TTBR)
```

The structure is deliberately the same as the other backends: the `ArchOps` impl is a thin delegation
layer, and the real per-operation code lives in a focused module, so a reader can find the ARM interrupt
masking in `irq_*`, the CPU id derivation from MPIDR in `cpu_id`, and the translation-table-base switch
in `address_space`.

## The ARM machinery

Beyond the eight primitives, the aarch64 tree (`src/arch/aarch64/mod.rs:17`) carries the modules the
platform needs:

```
  gic         the Generic Interrupt Controller (GICv3 SPIs)
  psci        Power State Coordination Interface (CPU on/off)
  mmu         the ARM page-table format and translation regime
  timer       the generic timer (the ArchOps time counter)
  exceptions  the exception vector table and handlers
  fpu         floating-point / SIMD state
  uart        the serial console
  context     task context save/restore
  security    the arch security state
```

These are the aarch64 analogues of the x86_64 machinery: the GIC plays the role of the IO-APIC and LAPIC
(and is the backend for the [hardware broker's](../subsystems/hardware-broker/irq.md) IRQ grants on ARM),
PSCI starts secondary CPUs where x86 uses the trampoline, and the MMU module implements the ARM
page-table format behind the same [paging](../subsystems/memory/paging-manager.md) manager.

## Maturity

It is worth being precise about status. aarch64 is *architecture-ready*, not production: it implements
`ArchOps` and the platform modules exist, so the kernel builds and links for the target, which is what
the fail-to-link discipline of the [boundary](boundary.md) guarantees. It has not been through the same
runtime bring-up and hardware validation as x86_64. The plan is x86_64 in production first, then aarch64
and riscv64 to architecture-ready, then QEMU bring-up, then hardware. This page documents the code that
exists; it does not claim aarch64 is a proven runtime target yet.

## Source

```
  src/arch/aarch64/abi/archops.rs   the ArchOps backend (delegating to the modules below)
  src/arch/aarch64/gic/, psci/, mmu/, timer/, exceptions/, uart/   the ARM machinery
```

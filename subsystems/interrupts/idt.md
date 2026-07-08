# The Interrupt Descriptor Table

Every trap the CPU can take, a divide error, a page fault, a timer tick, a syscall, enters
the kernel through one entry in the interrupt descriptor table. This page documents the
vector layout, how the table is built, the gate and stack assignments, and how it is loaded.
The code is under `src/interrupts/idt/`.

## The vector map

The 256 vectors are partitioned into fixed ranges (`src/interrupts/idt/vectors.rs`):

```
  0  .. 31    CPU exceptions        (divide error 0, page fault 14, double fault 8, ...)
  32 .. 47    legacy IRQ range      (irq_to_vector(n) = 32 + n)
  48 .. 0xEF  user-allocatable      (the dynamic vector pool)
  0x80        syscall gate          (the int 0x80 legacy path)
  0xFA .. 0xFF LAPIC vectors        (LINT1, LINT0, perf, thermal, error, spurious)
```

The named IRQ vectors sit in the legacy range: timer at 32, keyboard at 33, cascade at 34,
mouse at 44. `is_exception`, `is_irq`, and `is_user_allocatable` are the range predicates,
and `irq_to_vector` / `vector_to_irq` convert between an IRQ line and its vector. Two
classification tables travel with the map: `exception_has_error_code`, which marks the eight
exceptions the CPU pushes an error code for (double fault, invalid TSS, segment-not-present,
stack-segment, general protection, page fault, alignment check, control protection), and
`exception_is_fatal`, which marks the ones the kernel does not attempt to recover from.

## Building the table

The IDT is a single lazily-built global (`src/interrupts/idt/table.rs:25`), constructed once
in `build_idt` in three passes:

```
  build_idt():
      configure_exceptions(idt)     vectors 0..31
      configure_irqs(idt)           timer, keyboard, mouse, broker IRQs
      configure_syscall(idt)        vector 0x80, DPL = ring 3
```

`configure_syscall` is the one entry deliberately reachable from user mode: it sets the
descriptor privilege level to ring 3 so a capsule can issue the legacy `int 0x80`, whereas
every other gate is ring 0 and a user attempt to invoke it faults. `configure_irqs` also
installs the hardware-broker IRQ entries, the vectors a claimed device's line is routed to,
from `arch::interrupt::broker` (`table.rs:162`).

## Gates and stacks

An entry is an interrupt gate or a trap gate (`idt/entry.rs:21`), the difference being
whether the CPU clears the interrupt flag on entry; `EntryOptions` carries the gate type, the
privilege level, the present bit, and an optional IST index, and `validate_ist_index` bounds
the index to 0 through 6 while `validate_handler_address` rejects a null handler.

Six exceptions run on dedicated interrupt-stack-table stacks rather than the interrupted
stack, so a fault taken in a fragile window lands on known-good memory:

```
  #DB debug           DEBUG_IST     a #DB in the kernel-entry window
  NMI                 NMI_IST       nested NMIs
  #DF double fault    DF_IST        recover from a stack overflow
  #GP general prot.   GP_IST        a CPL=3 #GP cannot land on a torn TSS.RSP0
  #PF page fault      PF_IST        guard-page handling
  #MC machine check   MC_IST        critical hardware errors
```

The IST constants come from the [GDT](../smp/README.md) and are one-based hardware slots,
while the `x86_64` crate's `set_stack_index` is zero-based and adds one internally, which is
why each assignment subtracts one (`table.rs:54`). That off-by-one is deliberate and load
bearing: getting it wrong points an exception at the adjacent stack.

## The trampoline split

Not every exception is installed as a plain handler. The CPL=3-reachable exceptions, divide
error, debug, breakpoint, overflow, bound-range, invalid-opcode, stack-segment, general
protection, page fault, alignment check, and SIMD, are installed as naked assembly
trampolines by address (`set_handler_addr`), while the rest use the compiler's
`extern "x86-interrupt"` wrappers. The reason is `swapgs`: a handler that reads per-CPU state
through `gs` must run on the kernel GS base, and an exception entered directly from user mode
arrives on the user GS base. The [trampolines](trampolines.md) page covers that mechanism;
the table just points the user-reachable vectors at them.

## Loading

`load` (`idt/load.rs:23`) calls `IDT.load()` and records a loaded flag; the module also
re-exports the `enable_interrupts`, `disable_interrupts`, `without_interrupts`, and
`halt_loop` primitives the rest of the kernel uses to control the interrupt flag. The table
itself is immutable after construction, so loading it on each CPU installs the same vetted
set of gates.

## Source

```
  src/interrupts/idt/vectors.rs   the vector map and classification tables
  src/interrupts/idt/table.rs     build_idt, gate and IST assignment, the trampoline split
  src/interrupts/idt/entry.rs     GateType, EntryOptions, the validators
  src/interrupts/idt/load.rs      load and the interrupt-flag primitives
```

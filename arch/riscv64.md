# The riscv64 Backend

riscv64 is the third backend, architecture-ready like [aarch64](aarch64.md): it implements the full
`ArchOps` boundary and carries the RISC-V machinery (the PLIC, SBI, the MMU, the `mtime` timer), so the
kernel compiles and links for 64-bit RISC-V. This page documents it. The code is under
`src/arch/riscv64/`.

## The ArchOps implementation

The `Riscv64` backend (`src/arch/riscv64/abi/archops.rs:24`) implements the eight primitives by
delegation, the same shape as the other two backends:

```
  halt()                -> halt::halt()
  enable_interrupts()   -> irq_enable::enable()        (SIE in sstatus)
  disable_interrupts()  -> irq_disable::disable()
  interrupts_enabled()  -> irq_state::enabled()
  current_cpu_id()      -> cpu_id::current()           (the hart id)
  read_time_counter()   -> time::counter()             (mtime / the time CSR)
  flush_tlb_one(addr)   -> tlb::flush_one(addr)         (sfence.vma)
  switch_address_space  -> address_space::switch(root) (satp)
```

The CPU id is the hart id, the RISC-V hardware thread identifier, and the time counter reads the
`time` counter (the memory-mapped `mtime`). Address-space switch writes the `satp` register and the TLB
flush is `sfence.vma`, the RISC-V primitives corresponding to CR3/invlpg on x86 and TTBR/TLBI on ARM.

## The RISC-V machinery

The riscv64 tree (`src/arch/riscv64/mod.rs:17`) carries the platform modules:

```
  plic        the Platform-Level Interrupt Controller (external sources)
  sbi         the Supervisor Binary Interface (firmware calls: console, IPI, timer)
  mmu         the Sv39/Sv48 page-table format
  timer       the mtime counter (the ArchOps time counter)
  interrupts  the trap and interrupt handling
  fpu         floating-point state
  uart        the serial console
  context     task context save/restore
  security    the arch security state
```

The PLIC is the RISC-V interrupt controller and the backend for the
[hardware broker's](../subsystems/hardware-broker/irq.md) IRQ grants on RISC-V, and SBI is the interface
to the machine-mode firmware, used for the console, inter-hart interrupts, and the timer, where x86 would
use direct hardware access and ARM would use PSCI. The MMU module implements the RISC-V page-table format
behind the shared [paging](../subsystems/memory/paging-manager.md) manager.

## Maturity

Like aarch64, riscv64 is architecture-ready rather than production: the `ArchOps` impl and the platform
modules exist so the kernel builds and links, guaranteed complete by the fail-to-link discipline of the
[boundary](boundary.md), but it has not been through the runtime bring-up and validation that x86_64 has.
It follows the same plan: architecture-ready, then QEMU, then hardware. This page documents the code that
is there.

## Source

```
  src/arch/riscv64/abi/archops.rs   the ArchOps backend (delegating to the modules below)
  src/arch/riscv64/plic/, sbi/, mmu/, timer/, interrupts/, uart/   the RISC-V machinery
```

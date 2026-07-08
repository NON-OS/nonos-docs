# The ISR Trampolines

The trap handlers read per-CPU state through `gs`, and the kernel keeps a per-CPU base in
the kernel `GS` and the user's base in the user `GS`. An exception or IRQ taken directly from
a ring-3 capsule arrives with the user GS base loaded, so a handler that touched `gs`-relative
memory without switching first would fault on its own per-CPU access and storm. The naked
trampolines exist to close that window, and the timer trampoline additionally captures the
preempted capsule's full register state. The code is under `src/interrupts/isr/`.

## The swapgs pattern

The page-fault trampoline (`src/interrupts/isr/page_fault_trampoline.rs:30`) is the
representative shape. It inspects the saved `CS` on the interrupt stack frame to decide
whether the trap came from user mode, and only then swaps to the kernel GS base:

```
  test byte ptr [rsp + 16], 3     // saved CS: low two bits are the CPL
  jz  1f                          // came from ring 0 -> already on kernel GS
  swapgs                          // came from ring 3 -> switch to kernel GS
1:
  push all 15 GPRs
  ... set up args, fxsave, call the Rust handler, fxrstor ...
  pop  all 15 GPRs
  test byte ptr [rsp + 16], 3     // returning to ring 3?
  jz  2f
  swapgs                          // switch GS back
2:
  add rsp, 8                      // discard the CPU-pushed error code
  iretq
```

The `swapgs` is conditional on the interrupted privilege level in both directions, entry and
exit, because a trap taken from kernel mode is already on the kernel GS and a second swap
would be wrong. For a page fault the CPU pushes an error code, so the iretq frame sits one
word above it and the trampoline discards that word before `iretq`. The same pattern backs
the other CPL=3-reachable exceptions listed on the [IDT](idt.md) page.

## SIMD preservation

Between the pushes and the call, each trampoline reserves a 16-byte-aligned scratch area and
`fxsave`s the interrupted FPU and SSE state, restoring it with `fxrstor` after the handler
returns (`page_fault_trampoline.rs:54`). This matters because the Rust handler chain, and in
particular the BLAKE3 hashing the kernel does on some paths, clobbers the volatile XMM
registers; without the save, preempting a kernel-side SSE computation would corrupt it. The
save is on the handler path, not the fast return, so it is paid only when a trap is actually
serviced.

## The timer trampoline and preemption

The timer trampoline (`src/interrupts/isr/timer_trampoline.rs:65`) does everything the
exception trampolines do and one thing more: it lays the 15 saved GPRs plus the CPU-pushed
`rip/cs/rflags/rsp/ss` on the stack in exactly the layout of the first 160 bytes of
`UserContext`, then hands a pointer to that region to `timer_trap_handler`. When the trap came
from ring 3, the handler snapshots that frame onto the current process's `saved_user_context`
(`timer_trampoline.rs:151`):

```
  timer_trap_handler(ctx):
      if (ctx.cs & 3) == 3:                       // preempted a capsule
          pcb.saved_user_context = Some(snapshot of ctx)
      set_interrupt_context()
      stats::increment_timer()
      timer::on_timer_interrupt()                 // the tick body, scheduler hook
      process::exit::drain_pending_teardowns()
      drain_pending_kernel_stacks()
      send_eoi()
```

That snapshot is what makes preemptive multitasking work: the [scheduler](../scheduler/preemption.md)
resume hook later `take`s the most recent snapshot and `iretq`s back into the capsule through
`restore_user_context_iretq`, so a capsule interrupted mid-instruction resumes exactly where
it was. The register push order is chosen so the on-stack layout is byte-for-byte the leading
fields of `UserContext`; `rax` is pushed first and `r15` last, placing `r15` at offset zero.
The alignment is arranged so that after the 15 pushes and the `call` return address, `rsp` is
16-byte aligned at handler entry as the System V ABI requires.

## The end-of-interrupt

The tick body ends by acknowledging the interrupt to whichever controller is live
(`timer_trampoline.rs:198`): the LAPIC if it is enabled, otherwise the legacy PIC line. The
same choose-the-live-controller EOI is at the tail of every IRQ handler; the
[controllers](controllers.md) page covers the two paths. From ring 0 the CPU does not switch
to `TSS.RSP0`, so the trampoline runs on whatever kernel stack was current and skips both
swaps; the whole mechanism is a no-op overhead on a kernel-to-kernel tick.

## Source

```
  src/interrupts/isr/page_fault_trampoline.rs   the swapgs + fxsave pattern, error-code frame
  src/interrupts/isr/timer_trampoline.rs         the UserContext capture and preemption snapshot
  src/interrupts/isr/exception_trampoline.rs     the other CPL=3-reachable exception trampolines
  src/interrupts/isr/wrappers.rs                 the plain x86-interrupt wrappers for the rest
```

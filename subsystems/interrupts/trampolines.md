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
returns (`page_fault_trampoline.rs:58`). This matters because the Rust handler chain, and in
particular the BLAKE3 hashing the kernel does on some paths, clobbers the volatile XMM
registers; without the save, preempting a kernel-side SSE computation would corrupt it. The
save is on the handler path, not the fast return, so it is paid only when a trap is actually
serviced.

## The timer trampoline and preemption

The timer trampoline is a naked asm entry, `timer_trampoline`
(`src/interrupts/isr/timer_trampoline/trampoline.rs:18`), that does everything the exception
trampolines do and one thing more: it lays the 15 saved GPRs plus the CPU-pushed
`rip/cs/rflags/rsp/ss` on the stack in exactly the layout of the first 160 bytes of
`UserContext` (the 15 pushes at `trampoline.rs:31`, `fxsave` at `trampoline.rs:58`), then
hands a pointer to that region to the Rust body `timer_trap_handler`
(`handler.rs:34`). When the trap came from ring 3, the handler snapshots that frame onto the
current process's `saved_user_context` (`handler.rs:69`):

```
  timer_trap_handler(ctx):                          # handler.rs:34
      if (ctx.cs & 3) == 3:                          // preempted a capsule
          pcb.saved_user_context = Some(snapshot of ctx)   # :69
      set_interrupt_context()                        # :73
      stats::increment_timer()                       # :74
      send_eoi()                                     # :81  BEFORE the tick body, deliberately
      timer::on_timer_interrupt()                    // :82  the tick, may preempt-switch away
      if current_pid().is_some():                    # :90  never while a dying context is live
          process::exit::drain_pending_teardowns()
          drain_pending_kernel_stacks()
```

That snapshot is what makes preemptive multitasking work: the [scheduler](../scheduler/preemption.md)
resume hook later `take`s the most recent snapshot and `iretq`s back into the capsule through
`restore_user_context_iretq`, so a capsule interrupted mid-instruction resumes exactly where
it was. The register push order is chosen so the on-stack layout is byte-for-byte the leading
fields of `UserContext`; `rax` is pushed first and `r15` last, placing `r15` at offset zero.

## The end-of-interrupt

The EOI comes *before* the tick body, not after it, and the source is explicit about why
(`handler.rs:75`): `on_timer_interrupt` can preempt-switch away inside this interrupt, and a
deferred EOI would leave the timer vector in-service at the LAPIC, so no further ticks, sleep
wakeups, or preemption would fire until the preempted context happened to resume. Sending the
EOI at `handler.rs:81` is safe because the interrupt flag stays clear until the `iretq`, so the
handler cannot re-enter; the next tick simply pends and fires after the return. `send_eoi`
(`timer_trampoline/send_eoi.rs`) chooses the live controller, the LAPIC if enabled otherwise the
legacy PIC line, the same choice at the tail of every IRQ handler ([controllers](controllers.md)).
The two teardown drains run *after* the tick and only when a real process context is current
(`handler.rs:90`): draining while the interrupted context is a dying one would free the very
kernel stack the trap frame sits on, so the queues are retried on the next tick instead. From
ring 0 the CPU does not switch to `TSS.RSP0`, so the trampoline runs on whatever kernel stack was
current and skips both swaps; the whole mechanism is a no-op overhead on a kernel-to-kernel tick.

## Security analysis

The trampolines are the most safety-critical hand-written assembly in the kernel, because they run in
the exact window where the CPU has entered ring 0 but nothing has been made consistent yet. Three
properties rest on getting them right.

**The swapgs is conditioned on the interrupted CPL, both ways.** The entry `test byte ptr [rsp + 16], 3`
reads the saved `CS` and swaps to the kernel GS base only when the trap came from ring 3
(`page_fault_trampoline.rs:30`); the exit path tests again and swaps back only when returning to ring 3.
This is what makes the downstream `gs:8` per-CPU read on the [safety](safety.md) page correct. Get the
direction wrong and a kernel-to-kernel trap would swap to the user GS and every per-CPU access in the
handler would read attacker-influenced memory; that is why the condition is a raw CPL test on the trap
frame and not a flag the handler could have set. The user-reachable exceptions are installed as these
naked trampolines precisely so no compiler-generated prologue touches `gs` before the swap.

**The interrupted FPU and SSE state is preserved.** Each trampoline `fxsave`s into a 16-byte-aligned
scratch area before the call and `fxrstor`s after (`page_fault_trampoline.rs:58`). The concrete hazard is
that the Rust handler chain clobbers the volatile XMM registers (BLAKE3 hashing on some kernel paths uses
SSE), so preempting a kernel-side SSE computation without the save would corrupt it. The save is on the
serviced path, not the fast return, so a kernel-to-kernel tick pays nothing.

**The error-code frame is discarded exactly once.** For the exceptions the CPU pushes an error code, the
trampoline does `add rsp, 8` before `iretq` (`page_fault_trampoline.rs`), so the `iretq` frame is the
five CPU-pushed words and nothing else. This has to match the per-vector reality, an error-code vector
must discard and a non-error vector must not, and it is the same classification `exception_has_error_code`
encodes on the [IDT](idt.md) page. A mismatch would `iretq` off a misaligned frame and return to a
garbage `rip`. The honest boundary is that all of this is correctness maintained by hand: the hardware
does not check that the trampoline swapped, saved, or discarded correctly, so the trampoline is trusted
code and the security of everything above it is conditional on the trampoline being right.

## Debugging the trampolines

A trampoline bug does not usually print a clean message, because it corrupts the very state the handler
would use to report itself. The symptoms are indirect, and there are three canonical ones.

**A gs-fault storm.** If the entry swapgs is missing or mis-conditioned, the first `gs`-relative access
in the handler (the `mov {:e}, gs:8` in `set_interrupt_context`) faults, which re-enters through a
trampoline that also faults, and the machine wedges or triple-faults instead of printing. The tell is a
hang or reset immediately on the first trap from ring 3 with no `[TRAP …]` line at all, because the trap
machinery itself cannot run. Diagnosis is to confirm the vector is installed by address (a naked
trampoline) and not as a plain wrapper, since a wrapper reaches `gs` before any swap.

**A wrong return.** A misplaced error-code discard returns to a corrupted `rip`, which then faults
somewhere unrelated. The giveaway is a `[TRAP …]` line whose `rip=` bears no relation to any real code
address after a specific exception vector was taken; the `add rsp, 8` in that vector's trampoline is the
place to look.

**A silent SSE corruption.** If the `fxsave`/`fxrstor` pair is dropped, there is no fault at all, just a
kernel-side float computation that silently produces wrong bytes when a trap interleaves with it. This is
the hardest to catch because nothing crashes; it shows up as a nondeterministic hash or crypto mismatch
under interrupt load, and the fix is in the trampoline, not the crypto. The timer trampoline adds one
more thing to verify: because it lays the 15 GPRs plus the CPU frame in the exact leading layout of
`UserContext` (`timer_trampoline/trampoline.rs:31`), a wrong push order or a misaligned stack shows up as a capsule
that resumes at the wrong `rip` or with swapped registers after preemption, traced through the scheduler
resume hook on the [preemption](../scheduler/preemption.md) page.

## Source map

```
  src/interrupts/isr/page_fault_trampoline.rs   the swapgs + fxsave pattern, error-code frame
  src/interrupts/isr/timer_trampoline/           timer_trampoline (asm) + timer_trap_handler (UserContext capture, preemption snapshot)
  src/interrupts/isr/exception_trampoline.rs     the other CPL=3-reachable exception trampolines
  src/interrupts/isr/wrappers.rs                 the plain x86-interrupt wrappers for the rest
```

Every reference above is verified against those trees. The table that points the user-reachable vectors
at these trampolines is on the [IDT](idt.md) page, the per-CPU read the swapgs enables is on the
[safety](safety.md) page, and the `saved_user_context` the timer trampoline fills is resumed on the
[preemption](../scheduler/preemption.md) and [context switch](../process/context-switch.md) pages.

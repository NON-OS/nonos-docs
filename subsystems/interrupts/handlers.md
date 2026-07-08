# Exception and IRQ Handlers

Past the trampoline, each vector runs a Rust handler. The exception handlers decide whether a
fault is recoverable and, if not, whether it kills the offending capsule or halts the kernel;
the IRQ handlers run the device work and acknowledge the controller. This page documents both,
and the shared interrupt-context and end-of-interrupt handling. The code is under
`src/interrupts/handlers/`.

## The page fault

The page fault handler (`src/interrupts/handlers/exceptions/page_fault.rs:31`) is the busiest
and the most consequential, because demand paging routes every lazily-backed page through it.
It reads the faulting address from `CR2`, decodes the error code, and tries to handle the
fault before treating it as an error:

```
  handle(frame, error_code):
      addr = CR2
      increment page-fault stat
      if try_handle_fault(addr, error_code):  return   // handled, silent
      dump the trap, log it
      if fault came from user mode:  terminate_user_process()   // SIGSEGV-equivalent
      else:                          kernel_panic()             // halt
```

`try_handle_fault` (`page_fault.rs:60`) first asks the [hardening](../memory/hardening.md)
layer whether the address is a guard page, in which case it refuses to handle it and lets the
fault escalate, and otherwise hands it to the [paging manager](../memory/faults.md), which
performs demand backing or copy-on-write and returns success if it resolved the fault. A
handled demand fault returns silently and is not dumped, deliberately: dumping every lazily
backed page to the serial console would make a large allocation pathologically slow as it
faults in page by page. Only an unhandled fault is logged, and its disposition depends on
where it came from, a user-mode fault terminates that process with an error and yields, while
a kernel-mode fault is unrecoverable and halts. The logged address is passed through
`redact_address` so a fault log does not leak a raw kernel pointer.

## The double fault

The double fault handler (`exceptions/double_fault.rs:24`) runs on its own IST stack, since a
double fault often means the normal stack is unusable, and it does not attempt recovery: it
logs the frame and the error code, dumps the stack and instruction pointers (redacted), and
halts. A double fault is the CPU telling the kernel it could not deliver a prior fault, so
continuing is not meaningful; the handler's job is to leave a legible record and stop. The
remaining exception handlers, invalid TSS, segment-not-present, and the arithmetic and opcode
faults, follow the same shape at lower severity, classified by `exception_is_fatal`.

## The IRQ handlers

The IRQ handlers are lighter. Each sets the interrupt context, does its device work, updates a
stat, and sends the end-of-interrupt:

- **Timer** is the exception to "light": its body is `on_timer_interrupt`, which drives the
  100 Hz [scheduler tick](../scheduler/preemption.md), and it is reached through the
  [timer trampoline](trampolines.md) that also snapshots the preempted capsule.
- **Keyboard** and **mouse** (`handlers/irq/keyboard.rs:26`) set the interrupt context,
  increment their counter, and acknowledge. The microkernel build carries no in-kernel
  scancode pipeline, the PS/2 path lives in a driver capsule, so the kernel handler only has
  to keep the IDT slot resolvable and ack the line; the real input path is the
  [input subsystem](../input/README.md).
- **Syscall** (`handlers/irq/syscall.rs:19`) is the legacy `int 0x80` counter; the live
  syscall path is the `SYSCALL` instruction documented on the [syscall boundary](../syscall/boundary.md).

## Interrupt context and the end-of-interrupt

Two things are shared across every handler. First, each takes an interrupt-context guard at
entry via `set_interrupt_context`, which bumps a per-CPU nesting depth; the
[safety](safety.md) page covers what that guards against. Second, each ends by acknowledging
the interrupt to the live controller (`keyboard.rs:38`):

```
  send_eoi():
      if apic::is_enabled():  apic::send_eoi()          // preferred
      else:                   pic::send_eoi(irq_line)    // legacy fallback
```

The kernel runs on the local APIC when it is up and falls back to the 8259 PIC otherwise, and
every handler routes its acknowledgement through the same choice so the two controllers never
disagree about whether a line was serviced. The [controllers](controllers.md) page documents
both.

## Source

```
  src/interrupts/handlers/exceptions/page_fault.rs   demand/guard/terminate-vs-panic
  src/interrupts/handlers/exceptions/double_fault.rs the unrecoverable halt
  src/interrupts/handlers/exceptions/context.rs      ExceptionContext, error-code decode, logging
  src/interrupts/handlers/irq/                        timer, keyboard, mouse, syscall
```

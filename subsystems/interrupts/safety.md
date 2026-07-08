# Interrupt Safety

Two small mechanisms keep interrupt handling from tripping over itself: an RAII guard that
disables interrupts for a critical section and restores the prior state exactly, and a per-CPU
record of whether the CPU is currently inside an interrupt handler. This page documents both.
The code is under `src/interrupts/safety/`.

## The interrupt guard

`InterruptGuard` (`src/interrupts/safety/guard.rs:20`) is a scoped critical section. On
creation it reads the current interrupt-enable flag, disables interrupts if they were enabled,
and remembers what it found; on drop it re-enables them only if they had been enabled:

```
  InterruptGuard::new():
      was_enabled = interrupts_enabled()      // read IF from RFLAGS
      if was_enabled: cli
  Drop:
      if was_enabled: sti
```

Restoring the prior state rather than unconditionally enabling is what makes the guard safe to
nest: a guard taken inside another guard's section finds interrupts already disabled, records
that, and leaves them disabled on drop, so the outer section is not cut short. The flag is read
straight from `RFLAGS` with `pushfq`, and the enable and disable are `sti` and `cli`; the guard
restores state even on an unwinding path because it lives in `Drop`.

## Interrupt context

Separately, each handler records that its CPU is in interrupt context. `set_interrupt_context`
(`src/interrupts/safety/context.rs:74`) returns a guard that bumps a per-CPU nesting depth and
sets a per-CPU in-interrupt flag; the guard's `Drop` decrements the depth and clears the flag
only when the outermost handler exits:

```
  set_interrupt_context():
      depth[cpu] += 1
      in_interrupt[cpu] = true
      -> InterruptContext { cpu }
  Drop:
      if depth[cpu] drops to 0:  in_interrupt[cpu] = false
```

`in_interrupt_context()` lets code elsewhere ask whether it is running inside a handler, which
matters for choosing between a path that may sleep and one that must not. The state is
per-CPU, indexed by a CPU id read from the per-CPU data block at `gs:8` (the `cpu_id` field
sits just past the self-pointer at offset 0); the read requires the kernel GS base to be
loaded, which is exactly what the [trampolines](trampolines.md) guarantee before any Rust
handler runs. The depth counter, rather than a bare boolean, is what makes the flag correct
under nesting: a higher-priority interrupt taken inside a handler increments the depth, and the
flag stays set until the last one unwinds.

## Source

```
  src/interrupts/safety/guard.rs     InterruptGuard, the cli/sti critical section
  src/interrupts/safety/context.rs   per-CPU interrupt-context depth and flag, cpu_id via gs:8
```

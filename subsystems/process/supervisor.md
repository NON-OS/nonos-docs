# The Supervisor and the Reaper

Two things watch over process lifetimes, and they operate at different levels. The
init process, pid 1, is the userspace supervisor: it spawns every system capsule and
then runs a light residual loop, observing capsule liveness passively rather than
polling. The kernel reaper is the second half of exit: the preemption timer drains the
zombie pending-list and finalizes each dead process, reclaiming its memory. This page
documents both, and it completes the exit story the [lifecycle](lifecycle.md) page
began.

## Init, the userspace supervisor

`run_init` (`src/userspace/init/entry.rs:20`) is what pid 1 runs. It brings the whole
userland up in a fixed order and then hands off to its supervisor loop:

```
  run_init() -> !:
      spawn ramfs, then the core capsules that depend on it
      spawn the display core, the drivers, the vfs, the network stack
      spawn the desktop, the market, the apps
      lower_init_priority()      init drops to Priority::Low
      yield_after_spawns()       yield repeatedly to let them start
      launch_final_payload()
      init_loop()                the residual supervisor loop
```

The spawn order is a dependency order: the ramfs comes up first because later capsules
stage in it, the display core and drivers before the desktop that draws on them, the
network stack before the capsules that use it. Each is a [verified
spawn](../../security/capsules-and-trust.md). Once everything is running, init lowers
its own priority to `Low` so it never competes with the capsules it launched, yields to
let them initialise, and enters the loop.

## The init loop

`init_loop` (`src/userspace/init/supervisor/loop_impl.rs:25`) is deliberately light, and
its doc comment states the liveness philosophy exactly:

```
  init_loop() -> !:
      loop:
          if a second has passed since the last tick:
              services::lifecycle::tick()
          yield_now()
```

It ticks the service lifecycle registry once per second and otherwise yields. The key
design point is what it does not do: the kernel does not actively probe capsules for
liveness. A capsule that has exited is observed as dead on its next IPC, through the
process state machine that already tracks it, so there is no health-check traffic and no
polling thread. The supervisor's job is to walk the lifecycle registry on a slow tick,
not to interrogate every capsule.

## The reaper: the second phase of exit

When a process exits, [teardown](lifecycle.md) marks it a zombie, releases its broker
resources, takes it off the run queue, and enqueues its pid on a pending list
(`src/process/exit/pending.rs:18`). The pending list is drained and finalized by the
kernel reaper, and the reaper is driven by the preemption timer. On each tick the timer
interrupt calls `drain_pending_teardowns` (`src/interrupts/isr/timer_trampoline.rs:193`),
which is `drain`:

```
  drain():
      try_lock the pending list, else return           non-blocking in the ISR
      if empty, return
      take all pending pids
      for each: finalize_teardown(pid)
```

The `try_lock` matters: the reaper runs in the timer interrupt, so it must never block on
the pending-list lock, and if it cannot take it this tick it simply reaps on the next
one. Zombies are therefore finalized promptly, on the next timer tick after they are
enqueued, without a dedicated reaper thread.

## Finalizing a process

`finalize_teardown` (`src/process/exit/finalize.rs:11`) does the reclaim that teardown
deferred:

```
  finalize_teardown(pid):
      address_space::lifecycle::release(pcb)      free and zero the frames
      release broker resources for the pid         devices, IRQ, DMA, PIO (again)
      unregister the pid's service endpoints
      unregister the pid's IPC inbox
      clear its interrupt context and FPU state
      reparent_orphans(pid)
      PROCESS_TABLE.terminate_process(pid)          remove it from the table
```

The first step is where a capsule's memory is actually scrubbed: `release` clears the
VMAs and calls the ASID-scoped cleanup that frees the leaf frames and page tables, and
every freed frame is zeroed on the way out, which is the mechanism behind the
[ZeroState guarantee](../memory/zeroization.md). The broker release runs a second time
here, idempotently, so nothing the process held survives even if teardown was partial.
Its service endpoints and IPC inbox are unregistered so no message can be routed to a
dead process, its saved interrupt and FPU state are cleared, and finally its entry is
removed from the [process table](process-table.md).

## Reparenting orphans

Before the process leaves the table, `reparent_orphans(pid)` moves any children it still
has to a surviving parent, so a process that exits with live children does not leave them
pointing at a pid that is about to be freed. This is the standard orphan-reparenting a
process model needs, run at the moment the parent is finalized rather than left for the
children to discover.

## Source

```
  src/userspace/init/entry.rs                    run_init, the capsule spawn order
  src/userspace/init/supervisor/loop_impl.rs      the init supervisor loop
  src/process/exit/pending.rs                     the zombie pending-list and drain
  src/process/exit/finalize.rs                    finalize_teardown
  src/interrupts/isr/timer_trampoline.rs          the timer-driven reap
```

# Scheduler

The scheduler decides which capsule runs next. It is cooperative and preemptive
at once: capsules yield voluntarily at wait points, and a 100 Hz timer preempts a
capsule that overruns its slice. Selection is a fixed walk over five priority
classes. The [architecture overview](../architecture/overview.md) covers this in
section 11; here is the full machinery.

---

## Selection

`select_next_process` (`src/process/scheduler/selection/select.rs`) gathers the
runnable pids and walks five priority classes in a fixed order, taking the first
class that has a runnable process:

```
  RealTime  >  High  >  Normal  >  Low  >  Idle
```

Within a class, selection is round-robin. It remembers the last pid it scheduled
in `LAST_SCHEDULED_PID` and picks the next runnable pid after it, wrapping around,
so no process in a class is starved by its neighbours. If no class yields a pick,
a fallback selection runs over the runnable set.

```
  select_next_process()
    runnable = get_runnable_pids()
    if runnable is empty: return None       (the CPU idles)
    for prio in [RealTime, High, Normal, Low, Idle]:
        if pid = select_by_priority(runnable, last, current, prio):
            LAST_SCHEDULED_PID = pid
            return pid
    return select_fallback(runnable, current)
```

The runnable set is backed by a `VecDeque`
(`src/process/scheduler/dispatch/run_queue.rs`), appended at the back as
processes become ready and drained as they are picked. A separate deadline
module exists in the tree but is not wired into this selection path today; the
live policy is the five-class priority walk above.

## Preemption

A 100 Hz LAPIC timer drives preemption (`TICK_HZ = 100`,
`src/arch/x86_64/interrupt/apic/preemption/install.rs:25`). Each tick, ten
milliseconds apart, runs the tick handler:

```
  tick()                                  src/process/scheduler/preemption/tick.rs:21
    decrement the current time slice
    if it reached zero and preemption is enabled:
        set NEED_RESCHEDULE
    if any realtime task is runnable:
        set NEED_RESCHEDULE
```

`NEED_RESCHEDULE` is a flag the kernel checks at safe points; when set, it forces
a reschedule rather than letting the current capsule continue. A realtime task
becoming runnable sets it immediately, so a high-priority capsule does not wait
for a slice to expire.

## Yielding

A capsule yields voluntarily through `yield_now`
(`src/process/scheduler/preemption/yield_impl.rs:22`), which most blocking
syscalls call when they have nothing to do:

```
  yield_now()
    without_interrupts:
        save the current context (registers, FPU state)
        move the current process Running -> Ready, back onto the runqueue
        select_next_process: the priority walk above
        if next is a different pid: switch_to_process(next)
        else: stay Running
```

Yielding with interrupts disabled keeps the save-and-switch atomic against a
timer tick landing mid-switch. The heavy lifting is in `perform_yield_inline`
(`src/process/scheduler/preemption/yield_body.rs`), which saves context, requeues
the current pid, selects the next, and switches.

## Sleeping and waking

Blocking is not spinning. A capsule with nothing to do sleeps until a deadline or
an event (`src/process/scheduler/dispatch/sleep.rs`):

```
  sleep_until(pid, wake_ms)        record the wake time, state Sleeping,
                                   remove from the runqueue
  wake_process(pid)               clear the sleep, state Ready, back on the runqueue
  check_sleeping_processes()      wake every pid whose wake time has passed
```

This is the machinery underneath IPC blocking, the input router, and IRQ waits. A
capsule calling `MkIpcRecv` with no message waiting sleeps on a deadline and
yields; a delivery into its inbox calls `wake_process` and it becomes runnable
again. `check_sleeping_processes` is what turns a timed sleep back into a
runnable process when its deadline arrives.

## Process states

A process moves through a small set of states
(`src/process/core/types.rs`):

```
  New         created, not yet runnable
  Ready       runnable, on the runqueue
  Running     executing on a CPU
  Sleeping    waiting on a sleep deadline or an event
  Stopped     halted by a debugger or signal
  Zombie(c)   exited with code c, awaiting reap
  Terminated  reaped
```

A spawned capsule starts `Ready` (verified spawn adds it to the runqueue without
running it), becomes `Running` on its first-entry switch, cycles between
`Running`, `Ready`, and `Sleeping` over its life, and ends `Terminated`.

---

## First entry into a capsule

The transition into a freshly spawned capsule is a special case of switching.
Install left an `iretq` frame in the process control block. The dispatcher
(`src/arch/x86_64/context/switch/dispatch.rs:39`) checks for that pending entry
first; if present, it sets the kernel stack, loads the frame, and executes
`iretq` to ring 3 at the ELF entry point. If instead the process was blocked in a
syscall, its saved kernel context is resumed. If it has saved user registers,
those are restored. The fallback resumes a kernel thread. This ordering is what
lets the same switch path handle a brand new capsule, a capsule returning from a
blocking syscall, and a capsule preempted mid-execution.

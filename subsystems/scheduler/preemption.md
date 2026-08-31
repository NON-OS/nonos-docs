# Scheduler: Preemption

The scheduler is preemptive and cooperative at once. A process yields voluntarily at
natural wait points, and a periodic timer preempts one that overruns its slice. The
timer does not switch inline; it charges the running process's time slice and, when the
slice is spent or a realtime task is runnable, flags a reschedule that fires at the next
safe point. This page documents the tick, the time slice, the deferred-reschedule flag,
and the voluntary yield. The code is under `src/process/scheduler/preemption/`.

## The timer tick

The preemption timer fires at 100 Hz, one tick every 10 ms, and each tick runs `tick`
(`src/process/scheduler/preemption/tick.rs:22`):

```
  tick():                                              # tick.rs:22
      charge_tick(CURRENT_PID)
      scheduler tick_count += 1
      if spend_time_slice() == 1:                       # this CPU's own slice
          record a time-slice exhaustion
          if kernel_preempt() policy allows: set_reschedule()
      if has_realtime_tasks(): set_reschedule()
```

Every tick charges the running process's per-process tick accounting, bumps the global
tick count, and spends one tick of the current slice. Crucially the slice and the
reschedule flag are per-CPU state, not machine globals: `spend_time_slice`
(`state.rs:40`) decrements `smp::percpu::current().time_slice`, saturating at zero so it
never underflows, and returns the value before the decrement so the tick that took the
slice from one to zero is the one that exhausted it. This is a fix for a real bug (comment
`state.rs:23`): as single globals every CPU's timer tick decayed the same counter, so N
cores exhausted a slice N times too fast and a reschedule raised on any core was seen by
all. When the slice reaches zero the tick records the exhaustion and, if
`kernel_preempt()` permits, calls `set_reschedule()` (`state.rs:53`). Separately, if any
realtime task is runnable, the tick sets the reschedule flag regardless of the current
slice, so a realtime task does not have to wait for the running process's slice to expire.
On the shipping single-core build there is one CPU, so this per-CPU state behaves like a
global, but the wiring is per-CPU.

## The time slice

This CPU's `time_slice` (`smp::percpu::current().time_slice`) is the running process's
remaining ticks. It is reset to `DEFAULT_TIME_SLICE` (10, `state.rs:20`) via
`set_time_slice` (`state.rs:34`) when a process is dispatched, as the
[context switch](../process/context-switch.md) does on first entry, and counts down one per
tick. Ten ticks at 100 Hz gives a process up to 100 ms on the CPU before the timer considers
preempting it.
Because the slice is charged per tick rather than by wall-clock reading, a process that
blocks and yields before its slice expires simply gives up the rest of it, and the next
dispatch starts a fresh slice.

## Deferred reschedule

The tick runs in the timer interrupt, and it deliberately does not perform a context
switch there. Switching inside the ISR, in the middle of whatever the interrupted code
was doing, would be unsafe. Instead the tick sets this CPU's `need_resched` flag through
`set_reschedule` (`state.rs:53`) and returns, and the actual switch happens later, at a safe
return point where the kernel checks the flag with `need_reschedule` (`state.rs:49`) and
calls the scheduler. This split, decide-to-preempt in the tick, switch at a safe point, is
what keeps preemption from corrupting the state of the code it interrupts. The flag is a
`Release` store (`state.rs:54`) so the CPU that later reads it sees the decision.

## Voluntary yield

A process gives up the CPU voluntarily through `yield_now`
(`src/process/scheduler/preemption/yield_impl.rs:22`):

```
  yield_now():
      scheduler voluntary_yields += 1
      without_interrupts(|| contract_switch(SwitchIntent::Yield))
```

It records the voluntary yield and, with interrupts disabled across the whole operation,
invokes the scheduler switch contract with a `Yield` intent. Disabling interrupts is the
same discipline the paging and usercopy paths use: the save-select-switch sequence must
not be interrupted partway. This is the path underneath every natural wait point, an IPC
receive with nothing queued, a sleep, an IRQ wait, so a blocked capsule yields rather
than spins.

## The switch contract

Both the voluntary yield and the deferred preemption converge on the scheduler switch
contract (`src/process/scheduler/contract/`), invoked with a `SwitchIntent` that
distinguishes a voluntary yield from a forced preemption. The contract saves the current
process's context, calls [selection](selection.md) to choose the next pid, and performs
the [context switch](../process/context-switch.md) into it, or stays on the current
process if selection returns it. The intent lets the contract account for voluntary
versus involuntary switches, which show up as the `voluntary_switches` and
`involuntary_switches` counters on the PCB.

## Security analysis

Preemption runs in an interrupt, and it decides when to hand the CPU away, so its safety rests on not
switching where a switch would corrupt state and on not letting one process hold the CPU indefinitely.
Three properties hold.

**The tick never switches inline.** `tick` (`tick.rs:22`) charges accounting, decrements the slice, and
at most sets this CPU's `need_resched` with a release-ordered store via `set_reschedule` (`state.rs:54`);
it performs no context switch. Switching inside the ISR, in the middle of whatever the interrupted code was doing, would run the
scheduler over a half-updated kernel state, so the decision is deferred to a safe return point where the
kernel checks the flag. This is the discipline that keeps preemption from corrupting the code it
interrupts, and it is why the flag is a `Release` store: the CPU that later reads it sees a fully-formed
decision.

**A runaway process is always preempted.** The slice is a per-CPU countdown of ticks reset to
`DEFAULT_TIME_SLICE` (10, `state.rs:20`) on dispatch and saturated at zero by `spend_time_slice`
(`state.rs:40`) so it never underflows. When it reaches zero the tick records a
`time_slice_exhaustion` and, if `kernel_preempt()` policy allows, flags a reschedule. So a process that
never yields voluntarily still loses the CPU when its slice is spent: there is no way for a compute-bound
capsule to hold a core forever, which is the liveness property a preemptive scheduler owes the rest of
the system.

**Realtime work is not held hostage by the current slice.** If any realtime task is runnable, the tick
sets the reschedule flag regardless of the running process's remaining slice
(`tick.rs:33`, `has_realtime_tasks`), so a realtime task does not have to wait out a normal task's slice. The honest boundary:
whether an exhausted slice actually forces a switch is gated on `sys::policy::kernel_preempt()`, so the
policy can decline to preempt a kernel-mode path, and the switch only happens once execution reaches a
safe point that checks the flag, so a long non-preemptible kernel section defers the switch until it
returns. The voluntary `yield_now` disables interrupts across the whole save-select-switch
(`yield_impl.rs:22`), the same interrupts-off discipline the paging and usercopy paths use, so the
sequence is not interrupted partway.

## Debugging preemption

The two failure shapes are a process that hogs the CPU and a process that is preempted when it should not
be. A capsule that never yields the core is a `NEED_RESCHEDULE` that is set but never acted on: the tick
did its job (check that `time_slice_exhaustions` in `SCHEDULER_STATS` is climbing) but the safe-point
check that consumes the flag is not being reached, which on a wedged kernel path means the code never
returned to where the flag is read. If `time_slice_exhaustions` is not climbing, the timer is not
ticking at all, or this CPU's `time_slice` was never reset on dispatch and sits at zero doing nothing. A
realtime task that starts late despite being runnable is `has_realtime_tasks()` not reporting it, or the
reschedule flag being set but not consumed, the same safe-point question. The `voluntary_switches`
versus `involuntary_switches` counters on the PCB distinguish the two paths: a process accumulating only
involuntary switches is always being preempted rather than yielding, which is expected for a compute
capsule, while a process that should block but shows involuntary switches is failing to hit a voluntary
wait point and being timer-preempted out of a spin instead.

## Source map

```
  src/process/scheduler/preemption/tick.rs        the timer tick and the slice spend
  src/process/scheduler/preemption/yield_impl.rs   the voluntary yield
  src/process/scheduler/preemption/state.rs        per-CPU time_slice / need_resched accessors,
                                                   DEFAULT_TIME_SLICE, SCHEDULER_STATS
  src/smp/percpu/types.rs                          the per-CPU time_slice and need_resched fields
  src/process/scheduler/contract/                  the switch contract and SwitchIntent
```

Every reference above is verified against those trees. The slice reset on dispatch happens in the
[context switch](../process/context-switch.md); the selection the contract calls to pick the next pid is
on the [selection](selection.md) page; the wait points that lead into `yield_now` are on the
[sleep and wake](sleep-wake.md) page; and the `voluntary_switches` / `involuntary_switches` counters live
on the [PCB](../process/pcb.md).

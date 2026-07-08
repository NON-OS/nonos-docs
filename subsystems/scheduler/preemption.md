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
  tick():
      charge a tick against the current process's accounting
      scheduler tick_count += 1
      decrement CURRENT_TIME_SLICE (floored at zero)
      if the slice just reached zero:
          record a time-slice exhaustion
          if kernel_preempt() policy allows: NEED_RESCHEDULE = true
      if any realtime task is runnable: NEED_RESCHEDULE = true
```

Every tick charges the running process's per-process tick accounting, bumps the global
tick count, and decrements the current time slice. The slice is a countdown of ticks;
`fetch_update` floors it at zero so it never underflows. When the decrement takes the
slice to zero, the tick records the exhaustion and, if the preemption policy permits,
sets `NEED_RESCHEDULE`. Separately, if any realtime task is runnable, the tick sets
`NEED_RESCHEDULE` regardless of the current slice, so a realtime task does not have to
wait for the running process's slice to expire.

## The time slice

`CURRENT_TIME_SLICE` is the running process's remaining ticks. It is reset to
`DEFAULT_TIME_SLICE` when a process is dispatched, as the [context switch](../process/context-switch.md)
does on first entry, and counts down one per tick. A slice of, for example, ten ticks at
100 Hz gives a process up to 100 ms on the CPU before the timer considers preempting it.
Because the slice is charged per tick rather than by wall-clock reading, a process that
blocks and yields before its slice expires simply gives up the rest of it, and the next
dispatch starts a fresh slice.

## Deferred reschedule

The tick runs in the timer interrupt, and it deliberately does not perform a context
switch there. Switching inside the ISR, in the middle of whatever the interrupted code
was doing, would be unsafe. Instead the tick sets the `NEED_RESCHEDULE` flag and returns,
and the actual switch happens later, at a safe return point where the kernel checks the
flag and calls the scheduler. This split, decide-to-preempt in the tick, switch at a safe
point, is what keeps preemption from corrupting the state of the code it interrupts. The
flag is a release-ordered store so the CPU that later reads it sees the decision.

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

## Source

```
  src/process/scheduler/preemption/tick.rs        the timer tick and the slice
  src/process/scheduler/preemption/yield_impl.rs   the voluntary yield
  src/process/scheduler/preemption/state.rs        CURRENT_TIME_SLICE, NEED_RESCHEDULE
  src/process/scheduler/contract/                  the switch contract and SwitchIntent
```

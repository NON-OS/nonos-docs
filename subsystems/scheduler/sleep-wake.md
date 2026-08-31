# Scheduler: Sleep and Wake

A process that has nothing to do does not spin waiting for it. It sleeps: it comes off
the run queue, records when or why it should wake, and yields the CPU. It is woken
either by an explicit event, a message delivered or an interrupt fired, or by the timer
when a deadline it was waiting on passes. This is the machinery underneath IPC blocking,
IRQ waiting, and timed sleeps. This page documents the sleeping set, the wake-generation
counter that closes the lost-wakeup race, going to sleep, waking, and the timer-driven
deadline wake. The code is `src/process/scheduler/dispatch/sleep.rs`.

## The sleeping set

Sleeping processes are held in a map from pid to wake time
(`sleep.rs:28`):

```
  static SLEEPING_PROCESSES: spin::RwLock<BTreeMap<u32, u64>>    pid -> wake_time_ms
```

The value is the wall-clock millisecond a timed sleeper should wake. A process waiting on
an event rather than a deadline is still recorded here so the scheduler knows it is
asleep; its entry is removed when the event wakes it. The map is behind a reader-writer
lock, and every accessor also takes `disable_interrupts_guard()` while it holds the lock
(`sleep.rs:62`, `:76`, `:89`, `:112`, `:117`, `:132`). That is not decoration: the timer
tick sweeps this table from interrupt context (`check_sleeping_processes`), so a tick
landing in the middle of an update would otherwise spin on a lock the interrupted code
cannot release. The sweep itself already runs with interrupts off, so its own guards are
no-ops (comment `sleep.rs:23`).

## The wake-generation counter

The hard part of a sleep primitive is the lost-wakeup race: a receiver checks its queue,
finds it empty, and is about to park, when the message and its wake arrive in that gap.
If the wake is only a state transition it lands on a still-`Running` process as a no-op,
the receiver then parks, and it sleeps on a queue that already has data. NØNOS closes this
with a per-pid generation counter (`sleep.rs:47`):

```
  static WAKE_GENERATION: [AtomicU64; 1024]     one slot per pid, indexed pid % 1024
  wake_token(pid)   = wake_slot(pid).load(Acquire)              (sleep.rs:56)
  wake_process(pid) = wake_slot(pid).fetch_add(1, AcqRel); ...  (sleep.rs:90)
```

The protocol a blocking caller runs is: read `wake_token(pid)` **before** checking its
wait condition, check the condition (drain the inbox, read the IRQ seq), and if it must
block, call `sleep_until_unless_woken(pid, deadline, token)`. That function
(`sleep.rs:74`) re-reads the generation under one interrupts-off guard and refuses to
sleep if it moved:

```
  sleep_until_unless_woken(pid, wake_time_ms, token):
      irq off
      if wake_slot(pid).load(Acquire) != token: return      # a wake landed; do not park
      SLEEPING_PROCESSES.insert(pid, wake_time_ms)
      state = Sleeping
      remove from run queue
```

Because `wake_process` bumps the generation *unconditionally and first* (`sleep.rs:90`),
before it looks at the process state, a wake either lands before the token re-read (the
generation differs and the caller does not park) or after the transition to `Sleeping`
(the wake finds a genuinely sleeping process to move). There is no gap. This is the
primary blocking path in the kernel: `wake_token` + `sleep_until_unless_woken` are what
IPC receive (`src/syscall/microkernel/ipc/recv.rs:82`, `:99`, `:132`, `:153`),
`recv_from` (`recv_from.rs:118`, `:128`), and IRQ wait (`irq/wait.rs:55`, `:63`) run.

The table is a fixed 1024-slot atomic array rather than a map for a specific reason
(comment `sleep.rs:38`): `wake_process` runs from the timer sweep in interrupt context,
and inserting into a map can allocate, which would spin on the heap lock with interrupts
off and freeze the machine. The tradeoff is slot aliasing: two pids 1024 apart share a
counter. That is harmless in one direction only, and deliberately so, a shared bump can
make a sleeper refuse to park one loop iteration early, but it can never let a sleeper
sleep through a wake meant for it.

## Going to sleep

`sleep_until` (`sleep.rs:60`) is the unguarded form, used where the caller has no
condition to race against:

```
  sleep_until(pid, wake_time_ms):
      irq off
      SLEEPING_PROCESSES.insert(pid, wake_time_ms)
      set the process state to Sleeping
      remove it from the run queue
```

Three things happen together: the pid is recorded with its wake time, its state becomes
`Sleeping`, and it is taken off the [run queue](selection.md) so the selector will not
consider it. After this the process is invisible to scheduling until something wakes it,
which is what makes a blocked capsule cost nothing: it is not scanned, not dispatched,
and not consuming a slice. A caller that does have a condition to check against a
concurrent waker uses `sleep_until_unless_woken` instead, never the bare `sleep_until`.

## Waking

`wake_process` (`sleep.rs:87`) bumps the generation and then wakes only a process that is
actually asleep:

```
  wake_process(pid):
      irq off
      wake_slot(pid).fetch_add(1, AcqRel)          # unconditional, always
      woke = false
      if process state == Sleeping:
          state = Ready
          woke = true
      if woke:
          SLEEPING_PROCESSES.remove(pid)
          add to run queue
          scheduler wakeups += 1
```

The order matters and is not the intuitive one. The generation bump is unconditional and
comes first (`sleep.rs:90`); it is what a racing sleeper reads to learn the wake happened.
The removal from `SLEEPING_PROCESSES` and the run-queue add are *conditional* on the
target having genuinely been `Sleeping` (`sleep.rs:104`). This is deliberate (comment
`sleep.rs:99`): a wake aimed at a `Running` or `Ready` target must not strip the sleep
deadline of a sleep the target is about to enter, or that later sleep would be
unwakeable by the timer sweep. A spurious or duplicate wake of a process that is not
sleeping therefore bumps a counter and does nothing else, which is exactly what makes
wake safe to call from any path that might race another waker.

## The timer-driven deadline wake

Timed sleepers are woken by `check_sleeping_processes` (`sleep.rs:131`), which the
scheduler runs from the timer tick:

```
  check_sleeping_processes():
      irq off
      now = timestamp_millis()
      under the read lock, collect up to 64 pids whose wake_time <= now
      for each collected pid:
          SLEEPING_PROCESSES.remove(pid)        # spend the entry first
          wake_process(pid)
```

It snapshots the expired sleepers into a fixed 64-element array while holding only the
read lock (`sleep.rs:134`), releases the lock, then for each pid removes its entry
(`sleep.rs:153`) before calling `wake_process`. The explicit remove is there because the
deadline has already passed, so the entry is spent whether or not the wake transitions the
process (it may already be `Running` via an early-return path); leaving it behind would
make the sweep re-chew it every tick until the 64-slot budget is exhausted. The fixed
array means the scan allocates nothing, which suits a path the timer drives; the tradeoff
is that a burst of more than 64 sleepers expiring in the same instant is woken across
several calls rather than all at once, which the following ticks resolve.

## The two ways a process wakes

Putting it together, a sleeping process wakes one of two ways. An explicit event calls
`wake_process` directly: an [IPC](../ipc/README.md) send that delivers to a waiting receiver
wakes it, and an interrupt that a driver was waiting on wakes the driver. A deadline
wakes through `check_sleeping_processes`: a process that parked with a future time,
including a receive with a timeout, is woken when that time passes. Either way the
process returns to `Ready` and rejoins the run queue, and the [selector](selection.md)
picks it up on a later pass.

## Security analysis

Sleep and wake are the liveness path: a blocked capsule must cost nothing while asleep and must reliably
come back when its event arrives. The properties here are about not losing a wakeup and not corrupting
state by waking the wrong thing. Three hold.

**A sleeping process consumes no scheduling resource.** `sleep_until` (`sleep.rs:60`) and
`sleep_until_unless_woken` (`sleep.rs:74`) record the pid, set its state to `Sleeping`, and remove it from
the run queue, all under one interrupts-off guard, so the selector never considers it: it is not scanned,
not dispatched, and not charged a slice. This is the mechanism underneath every IPC-receive, IRQ-wait, and
timed sleep, so a capsule waiting on an event does not steal CPU from one doing work.

**The lost-wakeup race is closed, not left to the caller.** This is the property the earlier design left
open and the current code fixes. A blocking caller reads `wake_token(pid)` (`sleep.rs:56`) before it tests
its wait condition, and `sleep_until_unless_woken` (`sleep.rs:74`) refuses to park if the generation moved
since that read. Because `wake_process` bumps the generation unconditionally and first (`sleep.rs:90`), a
wake that races ahead of the park cannot be lost: it either bumps the counter before the token re-read, so
the caller does not sleep, or it arrives after the transition to `Sleeping`, so it finds a real sleeper.
The check and the transition are under a single guard, so there is no window between them. The honest
boundary is the 1024-slot aliasing (comment `sleep.rs:38`): two pids sharing a slot can make one refuse a
park one iteration early, wasteful but safe, and can never permit sleeping through a wake.

**Waking is idempotent and does not corrupt a non-sleeping target.** `wake_process` (`sleep.rs:87`) only
removes the sleeping entry and re-queues the pid when the target was genuinely `Sleeping` (`sleep.rs:104`);
a wake landing on a `Running` or `Ready` process bumps the generation and stops. This is what keeps an IPC
delivery and a timeout expiring at nearly the same instant from double-queuing a pid or stripping the
deadline of a sleep the target has not yet entered. The 64-entry cap on the deadline sweep (`sleep.rs:134`)
means a burst of more than 64 sleepers expiring in the same instant is spread over a few ticks, which the
following ticks resolve.

## Debugging sleep and wake

A process stuck asleep forever is the headline failure, and with the generation counter in place its
causes are narrower than they used to be. First rule out the mechanical ones: use `is_sleeping(pid)`
(`sleep.rs:111`) to confirm the process is actually in the sleeping set, and `get_remaining_sleep(pid)`
(`sleep.rs:116`) to read its wake time. A wake time in the past with the pid still sleeping means the
deadline scan is not running, which points at `check_sleeping_processes` not being reached from the timer
at all rather than at the sleep code. No entry at all means it already woke to `Ready` and the stall is in
selection starving it (see [selection](selection.md)).

The genuine lost wakeup, a waker signalling before the sleeper parks, is what the token protocol prevents,
so a stall that survives it is almost always a caller that did not follow the protocol: it must read
`wake_token` before checking its condition and pass that token to `sleep_until_unless_woken`, not call the
bare `sleep_until` after a check. A caller that reads the token too late, after draining, reintroduces the
gap the counter exists to close. The `wakeups` counter in `SCHEDULER_STATS` climbing without the target
becoming `Ready` means wakes are landing on non-`Sleeping` targets: with the token protocol that is benign
(the sleeper saw the generation move and did not park), but a run of it alongside a stalled pid is worth
correlating against whether that pid's blocking loop reads the token in the right order.

## Source map

```
  src/process/scheduler/dispatch/sleep.rs    the sleeping set, WAKE_GENERATION, wake_token,
                                              sleep_until, sleep_until_unless_woken, wake_process,
                                              check_sleeping_processes
  src/process/scheduler/dispatch/wakeup.rs   wakeup(): a stats bump that calls
                                              check_sleeping_processes and set_reschedule
  src/syscall/microkernel/ipc/recv.rs        the token-protocol caller for IPC receive
  src/syscall/microkernel/irq/wait.rs        the token-protocol caller for IRQ wait
```

Every reference above is verified against those trees. The run queue a woken process rejoins and the
selector that picks it up are on the [selection](selection.md) page; the timer that drives
`check_sleeping_processes` is the same tick documented on the [preemption](preemption.md) page; the
`Sleeping` and `Ready` states these transitions move between are on the
[lifecycle](../process/lifecycle.md) page.

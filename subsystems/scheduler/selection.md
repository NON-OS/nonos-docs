# Scheduler: Selection

The scheduler chooses the next process to run by walking five priority bands in
strict order and rotating round-robin within a band. Higher-priority work always
runs before lower, and within a band no process starves its neighbours. This page
documents the run queue the candidates come from, the priority-band walk, the
round-robin rule inside a band, and the fallback that keeps the current process
running when nothing else is ready. The code is under `src/process/scheduler/`.

## The run queue

Runnable processes are held in a first-in-first-out queue of pids
(`src/process/scheduler/dispatch/run_queue.rs:25`):

```
  static PID_RUN_QUEUE: Mutex<VecDeque<u32>>

  add_to_run_queue(pid)         push to the back, refusing duplicates
  add_to_run_queue_front(pid)   push to the front, refusing duplicates
  remove_from_run_queue(pid)    remove by value
  get_runnable_pids()           a snapshot for the selector to scan
```

The queue is a `VecDeque` ordered by arrival, and the source states why: FIFO by
arrival means a long-running pid does not starve newcomers the way a pid-sorted set
would. Both insert paths refuse duplicates, so the same pid cannot be enqueued twice
and appear runnable in two places. A process becomes runnable when it is created, woken
from sleep, or made ready after preemption, and it leaves the queue when it blocks or
exits.

## The priority bands

`select_next_process` (`src/process/scheduler/selection/select.rs:26`) takes a snapshot
of the runnable pids and walks the five bands in order:

```
  select_next_process():
      current = CURRENT_PID (this CPU)
      runnable = get_runnable_pids(), else None if empty
      for band in [RealTime, High, Normal, Low, Idle]:
          if select_by_priority(runnable, band_last, current, band) is Some(pid):
              record it as this band's last and the overall last, return pid
      select_fallback(runnable, current)
```

The order is strict: `RealTime` before `High` before `Normal` before `Low` before
`Idle`. The walk takes the first band that has a runnable process and never looks at a
lower band while a higher one has work, so a realtime process always preempts a normal
one, and idle-priority work runs only when nothing else can. Each band remembers the
last pid it scheduled in `LAST_PER_BAND[idx]`, and the overall last is `LAST_SCHEDULED_PID`.

## Round-robin within a band

`select_by_priority` (`select.rs:48`) is the round-robin. Within a single band it picks
the next pid after the one the band scheduled last, wrapping to the lowest when it runs
off the end:

```
  select_by_priority(pids, last, current, band):
      lowest = None; after = None
      for pid in pids, skipping current:
          skip unless the process is Ready and in this band
          lowest = min(lowest, pid)
          if pid > last: after = min(after, pid)
      return after or lowest
```

`after` is the smallest runnable pid greater than the band's last-scheduled pid, which
is the next one in rotation; when there is no such pid, the band has wrapped and it
falls back to `lowest`, the smallest pid in the band. Rotating by pid this way means
every ready process in a band takes a turn before any repeats, so no process in a band
is starved by its neighbours. The selector skips `current` deliberately, so it rotates
to another process rather than immediately reselecting the one that just ran.

## The fallback

If no band yields a candidate, `select_fallback` (`select.rs:70`) keeps the current
process running, but only if it is genuinely still runnable:

```
  select_fallback(pids, current):
      if current is not in the runnable set -> None
      if current's state is Ready           -> current
      else                                   -> None
```

This covers the case where the only runnable process is the one already on the CPU: it
continues rather than the scheduler idling. If even the current process is no longer
`Ready`, selection returns `None`, and the caller idles the CPU until an interrupt makes
something runnable.

## What is and is not the live policy

The five-band priority walk above is the scheduling policy in effect. The tree also
contains a `deadline` module (`src/process/scheduler/deadline/`) and a `realtime` module
(`src/process/scheduler/realtime/`), but the live selection path is the priority walk in
`select.rs`, not a deadline scheduler, so this page documents the policy that actually
runs. The [preemption](preemption.md) page covers the timer that forces a reselection,
and the [sleep and wake](sleep-wake.md) page covers how a process leaves and re-enters
the run queue.

## Source

```
  src/process/scheduler/selection/select.rs   select_next_process and the band walk
  src/process/scheduler/dispatch/run_queue.rs  the FIFO runnable queue
```

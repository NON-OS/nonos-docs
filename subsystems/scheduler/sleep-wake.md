# Scheduler: Sleep and Wake

A process that has nothing to do does not spin waiting for it. It sleeps: it comes off
the run queue, records when or why it should wake, and yields the CPU. It is woken
either by an explicit event, a message delivered or an interrupt fired, or by the timer
when a deadline it was waiting on passes. This is the machinery underneath IPC blocking,
IRQ waiting, and timed sleeps. This page documents the sleeping set, going to sleep,
waking, and the timer-driven deadline wake. The code is
`src/process/scheduler/dispatch/sleep.rs`.

## The sleeping set

Sleeping processes are held in a map from pid to wake time
(`src/process/scheduler/dispatch/sleep.rs:22`):

```
  static SLEEPING_PROCESSES: RwLock<BTreeMap<u32, u64>>    pid -> wake_time_ms
```

The value is the wall-clock millisecond a timed sleeper should wake. A process waiting on
an event rather than a deadline is still recorded here so the scheduler knows it is
asleep; its entry is removed when the event wakes it. The map is behind a reader-writer
lock, so the common read, the timer scanning for expired sleepers, does not block other
readers.

## Going to sleep

`sleep_until` (`sleep.rs:24`) puts a process to sleep:

```
  sleep_until(pid, wake_time_ms):
      SLEEPING_PROCESSES.insert(pid, wake_time_ms)
      set the process state to Sleeping
      remove it from the run queue
```

Three things happen together: the pid is recorded with its wake time, its state becomes
`Sleeping`, and it is taken off the [run queue](selection.md) so the selector will not
consider it. After this the process is invisible to scheduling until something wakes it,
which is what makes a blocked capsule cost nothing: it is not scanned, not dispatched,
and not consuming a slice.

## Waking

`wake_process` (`sleep.rs:33`) is the reverse, and it is careful to wake only a process
that is actually asleep:

```
  wake_process(pid):
      remove the pid from SLEEPING_PROCESSES
      if the process state is Sleeping:
          set it to Ready
          add it back to the run queue
          scheduler wakeups += 1
```

It removes the sleeping entry, and only if the process is genuinely in the `Sleeping`
state does it flip it to `Ready`, put it back on the run queue, and count the wakeup. The
state check means a spurious or duplicate wake of a process that is already running or
ready does nothing, so waking is safe to call from any path that might have raced with
another waker.

## The timer-driven deadline wake

Timed sleepers are woken by `check_sleeping_processes` (`sleep.rs:68`), which the
scheduler runs from the timer:

```
  check_sleeping_processes():
      now = timestamp_millis()
      under the read lock, collect up to 64 pids whose wake_time <= now
      for each collected pid: wake_process(pid)
```

It snapshots the expired sleepers into a fixed 64-element array while holding only the
read lock, then releases the lock and wakes each, so it never calls `wake_process`, which
takes the write lock, while still holding the read lock. The fixed array means the scan
allocates nothing, which suits a path the timer drives; the tradeoff is that a burst of
more than 64 sleepers expiring in the same instant is woken across several calls rather
than all at once, which the following ticks resolve.

## The two ways a process wakes

Putting it together, a sleeping process wakes one of two ways. An explicit event calls
`wake_process` directly: an [IPC](../ipc/README.md) send that delivers to a waiting receiver
wakes it, and an interrupt that a driver was waiting on wakes the driver. A deadline
wakes through `check_sleeping_processes`: a process that called `sleep_until` with a
future time, including a receive with a timeout, is woken when that time passes. Either
way the process returns to `Ready` and rejoins the run queue, and the
[selector](selection.md) picks it up on a later pass.

## Source

```
  src/process/scheduler/dispatch/sleep.rs    the sleeping set, sleep_until, wake_process,
                                              check_sleeping_processes
  src/process/scheduler/dispatch/wakeup.rs   wakeup helpers
```

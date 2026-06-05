# Input

Input is one of the clearest end-to-end paths in the system: a key press or mouse
move travels from a driver capsule, through one kernel ring, to a single router
capsule, and out over IPC to whoever is listening. The
[architecture overview](../architecture/overview.md) covers it in section 13; here
is the full path with sources.

---

## The event

An input event is a fixed struct
(`src/kernel_core/surface_registry/types.rs:44`):

```
  InputEvent
    kind          u16    key, pointer, touch, and so on
    flags         u16    modifier and state flags
    code          u32    key code or button id
    x             i32    absolute X
    y             i32    absolute Y
    delta_x       i32    relative X movement
    delta_y       i32    relative Y movement
    timestamp_ns  u64    nanosecond timestamp
```

It carries both absolute coordinates and relative deltas, so a pointer device that
reports motion and a touch device that reports position both fit one struct.

## The ring

The kernel owns one multi-producer single-consumer ring
(`src/kernel_core/surface_registry/input_ring.rs`), capacity 1024
(`INPUT_RING_CAP`). Many driver capsules post into it; exactly one router capsule
drains it. The ring is guarded by a short-held lock, plus a release-ordered
sequence counter and a single parked-waiter slot:

```
  RING      a 1024-entry buffer with head and tail, under a Mutex
  SEQ       a counter bumped once per posted event
  WAITER    the pid of the router parked on the ring, or 0
  DROPPED   a counter of events dropped when the ring was full
```

---

## Posting

A driver posts with `MkInputEventPost`, which lands in `post_input`
(`src/kernel_core/surface_registry/input_ring.rs:56`):

```
  post_input(ev)
    lock the ring
    if the next head would meet the tail: DROPPED += 1, return OutOfSlots
    write ev at head, advance head
    unlock
    SEQ.fetch_add(1, Release)
    if a waiter is parked: wake_process(waiter)
```

The lock is held only long enough to copy one event into the buffer. The sequence
bump and the wake happen after the lock is dropped, so a posting driver does not
hold the ring while it schedules the router. If the ring is full the event is
counted as dropped rather than blocking the driver, which keeps a fast input
source from stalling on a slow consumer. Posting requires the `InputSource`
capability.

## Waiting and draining

The router blocks until there is something to do, then takes a batch
(`src/syscall/dispatch/router/input_ops.rs`):

```
  MkInputEventWait(last_seq, timeout)
    arm_input_waiter(pid)            park this pid in WAITER
    if SEQ moved past last_seq or the timeout elapsed:
        clear_input_waiter(); return the new seq
    sleep_until(deadline); yield      and re-check on wake

  MkInputEventDrain(out, max)
    drain up to min(max, 64) events from the ring into out      MAX_DRAIN = 64
    return the count
```

`MkInputEventWait` is the parked side of the `post_input` wake: the router arms
itself as the waiter, sleeps, and is woken when a driver posts. `drain_input`
(`input_ring.rs:95`) copies out as many events as fit, up to the batch cap of 64,
advancing the tail. The wait and drain calls are gated by the `IPC` capability;
in practice only the one router capsule uses them.

---

## The full path

```
  driver capsules                kernel input ring              input router
   kbd / mouse / hid                (MPSC, cap 1024)
        |                                                          |
        |  MkInputEventPost(ev)                                    |
        +----------------->  post_input: push, SEQ++, wake --------+ wake_process
                                                                   |
                                          MkInputEventWait: parks  |  arm_input_waiter
                                            until SEQ advances     |  sleep_until
                                          MkInputEventDrain: batch |  up to 64
                                            of events              |
                                          parse pointer and key,   |
                                          forward over IPC to the   v
                                          desktop shell
```

The router is the single point that fans one shared ring out to subscribers. A
driver never needs to know who is listening; it posts, and the router decides
where events go. Today the router turns raw events into pointer and key events and
forwards them to the desktop shell over IPC ([ipc](ipc.md)), which repaints the
cursor and routes keystrokes to the focused window. Adding a new input device is a
new driver capsule that posts into the same ring; nothing downstream changes.

## Why one ring and one router

Routing input through a single kernel ring and a single userspace router keeps the
kernel's role minimal: it owns a bounded buffer and a wake, nothing more. All
policy, which window has focus, how a pointer delta becomes an absolute position,
how a key maps to a character, lives in the router and the shell, in user mode,
where it can be replaced without touching the kernel. The kernel guarantees only
that posted events are delivered in order to the one consumer, or counted as
dropped if that consumer falls too far behind.

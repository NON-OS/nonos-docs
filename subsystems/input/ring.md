# The Input Ring

All input in NØNOS funnels through one bounded ring in the kernel. Driver capsules post events
into it; a single router capsule drains it. The kernel owns only this ring and the wakeup; the
policy of routing events to windows lives in the router capsule. This page documents the ring. The
code is `src/kernel_core/surface_registry/input_ring.rs`.

## A bounded MPSC ring

The ring is multi-producer, single-consumer: many driver capsules (keyboard, mouse, touch) post,
and one `input_router` capsule drains. It is a fixed array behind a mutex (`input_ring.rs:27`):

```
  struct Ring { head, tail, buf: [InputEvent; INPUT_RING_CAP] }   // INPUT_RING_CAP = 1024
```

Both posting and draining take the mutex, but each critical section is short (a post writes one
event and advances one index), so the lock is held briefly. The single-consumer side means the
router capsule does its own per-source fan-out after draining, keeping that policy out of the
kernel.

## Posting

`post_input` (`input_ring.rs:55`) enqueues one event, or drops it if the ring is full:

```
  post_input(ev):
      lock ring
      next = (head + 1) % CAP
      if next == tail:  DROPPED += 1; return OutOfSlots   // full: drop, do not block
      buf[head] = ev; head = next
      unlock
      SEQ += 1                                            // publish a new sequence
      if a waiter is armed:  wake it
```

A full ring drops the event and bumps a `DROPPED` counter rather than blocking the producer: a
driver posting from an interrupt-driven path must never stall, so back-pressure is a visible drop
count, not a hang. After a successful post the global sequence number is incremented and, if the
router capsule has armed itself as a waiter, it is woken. The drop counter makes input loss
observable rather than silent.

## The sequence number and the waiter

The ring exposes a monotonic sequence number (`SEQ`) that increments on every post, and a single
waiter slot (`input_ring.rs:80`):

```
  arm_input_waiter(pid)    the router registers itself to be woken on the next post
  input_seq() -> u64       the current sequence
  clear_input_waiter()     deregister
```

The sequence number is what lets the consumer wait on an edge without missing events: it records
the sequence it last saw, arms itself as the waiter, and re-checks; if the sequence advanced, there
is new input to drain. A post wakes exactly the one armed waiter (`wake_process(waiter)`), which is
the single router capsule, through the [scheduler](../scheduler/sleep-wake.md). There is one waiter
slot because there is one consumer.

## Draining

`drain_input` (`input_ring.rs:95`) copies as many queued events as fit into the caller's buffer and
advances the tail:

```
  drain_input(out):
      lock ring
      while out has room and tail != head:
          out[n++] = buf[tail]; tail = (tail + 1) % CAP
      return n
```

Draining is bounded by the caller's buffer, so the consumer pulls a batch, processes it, and comes
back. Nothing in the kernel interprets the events; they are copied out verbatim for the router to
classify.

## Source

```
  src/kernel_core/surface_registry/input_ring.rs   the ring, post, drain, sequence, waiter
  src/kernel_core/surface_registry/types.rs         InputEvent and INPUT_RING_CAP
```

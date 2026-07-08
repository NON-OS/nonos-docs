# The Event and the Path

An input event travels from a driver capsule, through the kernel ring, to the router capsule, and
on to whichever consumer owns the focus. This page documents the event structure, the three
syscalls that move it, and the capsule path. The code is `src/syscall/dispatch/router/input_ops.rs`
and `src/userspace/capsule_input_router/`.

## The event

An `InputEvent` (`src/kernel_core/surface_registry/types.rs:44`) is a fixed, flat record:

```
  struct InputEvent {
      kind:    u16,    // key, pointer motion, button, ...
      flags:   u16,
      code:    u32,    // key code / button id
      x, y:    i32,    // absolute position
      delta_x, delta_y: i32,   // relative motion
      timestamp_ns: u64,
  }
```

It is a plain value type with no pointers, so it copies across the syscall boundary by value and
carries everything a consumer needs to interpret one event: the kind, a code, an absolute position,
a relative delta, and a timestamp. The kernel does not interpret any of these fields; it moves the
record.

## The three syscalls

Input is three `MkInputEvent*` syscalls, dispatched by `input_ops::handle`
(`src/syscall/dispatch/router/input_ops.rs:36`):

```
  MkInputEventPost(ev_ptr)                     a driver posts one event
  MkInputEventDrain(out_ptr, max)              the router drains up to max (<= 64) events
  MkInputEventWait(last_seq, timeout, out_ptr) block until the sequence advances or times out
```

`do_post` reads the event out of user memory and enqueues it, returning `ENOMEM` if the ring is
full. `do_drain` copies up to `MAX_DRAIN` (64) events into a kernel scratch buffer and then out to
the caller with the [usercopy](../memory/usercopy.md) checks, returning the count. `do_wait`
(`input_ops.rs:83`) is the edge-triggered wait: it arms the caller as the ring's waiter, compares
the current sequence to the `last_seq` the caller passed, and if it advanced (or the timeout
elapsed) writes the new sequence back and returns; otherwise it sleeps until a deadline and
re-checks. Each call is marked audited, and each validates its user buffer before touching it.

## The capsule path

The full path is capsule to capsule, with the kernel as the bounded rendezvous:

```
  driver capsule            kernel ring              input_router capsule       consumers
  (kbd / mouse / xhci)                                                          (shell / gui)
  MkInputEventPost   -->  post_input, wake  -->  MkInputEventWait + Drain  -->  per-source fan-out
```

A driver capsule that owns its device through the [hardware broker](../hardware-broker/irq.md)
translates hardware input into `InputEvent`s and posts them. The single `capsule_input_router`
(`src/userspace/capsule_input_router/`) waits on the sequence, drains the batch, and routes each
event to the consumer that should receive it, the focused window's shell or GUI capsule, over
[IPC](../ipc/README.md). The kernel's role is deliberately minimal: a bounded ring, a sequence
number, and one wakeup. The authority to post or drain is gated by the capability the
`MkInputEvent*` syscalls require, defined in the
[capability model](../../security/capabilities-and-tokens.md), so an arbitrary capsule cannot
inject synthetic input or eavesdrop on the stream.

## Source

```
  src/kernel_core/surface_registry/types.rs         InputEvent
  src/syscall/dispatch/router/input_ops.rs           the three MkInputEvent* handlers
  src/userspace/capsule_input_router/                the router capsule
```

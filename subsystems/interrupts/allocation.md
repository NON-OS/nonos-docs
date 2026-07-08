# Vector Allocation

The fixed vectors, the exceptions, the legacy IRQs, the syscall gate, are assigned at build
time. The rest of the vector space is a pool the kernel hands out at runtime, for example when
the hardware broker routes a claimed device's line to a fresh vector. This page documents that
pool. The code is under `src/interrupts/allocation/`.

## The registry

The allocator's state is one global registry (`src/interrupts/allocation/registry.rs:21`),
two parallel arrays indexed by vector:

```
  struct Registry {
      reserved: [bool; 256],                    // is this vector taken
      handlers: [Option<NoErrorHandler>; 256],  // its handler, if registered
  }
```

A vector is available only if it is neither reserved nor has a handler. The registry is behind
an `RwLock`, so allocation takes the write lock and the availability query takes the read lock.

## What is reserved

`init` (`allocation/init.rs:20`) marks the fixed part of the space as reserved before any
dynamic allocation can happen:

```
  reserve vectors 0 .. RESERVED_VECTORS_END (32)   // all CPU exceptions
  reserve TIMER_VECTOR    (32)
  reserve KEYBOARD_VECTOR (33)
  reserve SYSCALL_VECTOR  (0x80)
```

The first thirty-two vectors are the CPU exceptions and are never allocatable. The timer,
keyboard, and syscall vectors are individually reserved on top, because they have fixed
handlers installed in the [IDT](idt.md) and must not be handed to a dynamic requester.

## Allocating and freeing

`allocate_vector` (`allocation/allocator.rs:20`) walks upward from the first non-reserved
vector and claims the first free slot:

```
  allocate_vector():
      for vector in RESERVED_VECTORS_END ..= 255:
          if not reserved[vector] and handlers[vector] is None:
              reserved[vector] = true
              return Some(vector)
      None                                  // pool exhausted
```

`free_vector` refuses to free anything below `RESERVED_VECTORS_END` and refuses to free a
vector that was not allocated, then clears both the reserved flag and the handler. The
allocation starts at 32 rather than at the user-allocatable range's nominal start, so the
legacy IRQ vectors are claimable by a requester that owns the corresponding line, while the
individually-reserved timer, keyboard, and syscall vectors within that range stay off limits.
The failure modes are explicit, `None` when the pool is exhausted and an error string when a
free is invalid, rather than silent wraparound.

## Source

```
  src/interrupts/allocation/registry.rs   the reserved and handler arrays
  src/interrupts/allocation/init.rs        reserving the fixed vectors
  src/interrupts/allocation/allocator.rs   allocate_vector, free_vector, is_vector_available
  src/interrupts/allocation/handlers.rs    register_handler, get_handler, unregister_handler
```

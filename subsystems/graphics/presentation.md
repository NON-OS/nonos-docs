# Presentation and Vsync

Once a surface is drawn, it is presented, and a capsule that animates paces itself to the display's
refresh by waiting for vertical blank. The vsync path is worth documenting carefully because its
timing model was the difference between a smooth desktop and a slow one. This page documents present
and vsync. The code is `src/syscall/dispatch/router/surface_ops.rs` and
`src/kernel_core/surface_registry/vsync.rs`.

## The surface syscalls

Six `MkSurface*` and `MkDisplay*` syscalls make up the surface surface, dispatched by
`surface_ops::handle` (`surface_ops.rs:31`):

```
  MkSurfaceRegister   register a surface, return sid + handle
  MkSurfaceShare      mark it shareable (owner)
  MkSurfaceAttach     map it into the caller, return the descriptor + VA
  MkSurfaceRelease    drop a reference
  MkSurfacePresent    commit the surface's current contents
  MkDisplayVsyncWait  block until the next vertical blank
```

`map_err` (`surface_ops.rs:51`) maps the registry errors to errnos, so a bad handle or a non-owner
operation becomes `EINVAL` or `EPERM` for the caller. Present commits the owner's surface for
display; the compositor capsule reads the shared surfaces and builds the final image, and the
kernel's role is to have kept the frames coherent across the sharers.

## The vsync phase grid

`wait_for_vsync` (`vsync.rs:38`) blocks a caller until the next vblank, computed on a fixed phase
grid derived from absolute time:

```
  wait_for_vsync(display_id, pid):
      period   = 1e9 / target_hz          // 60 Hz default -> ~16.67 ms
      now      = time::now_ns()
      deadline = (now / period + 1) * period   // next boundary on the shared grid
      while now_ns() < deadline:
          sleep_until(deadline); yield
      publish LAST_VBLANK_NS = max(LAST_VBLANK_NS, deadline)
      return deadline
```

The deadline is quantized to a grid of period-sized slots anchored to absolute time, so every
capsule that waits within the same frame computes the *same* next boundary and wakes together. This
is the fix the code documents in place: an earlier version advanced one shared running deadline by a
full period per waiter, so with the compositor, window manager, shell, and cursor all waiting, each
was pushed a period past the last, and the effective refresh collapsed to target_hz divided by the
number of waiters, which is what made the desktop feel slow. The published `LAST_VBLANK_NS` is now
only a monotonic timestamp for readers (a `fetch_max`) and does not feed back into the next
deadline, so waiters can never serialize behind one another. The wait uses the monotonic
[clock](../time-and-clock/time-bases.md) and the [scheduler](../scheduler/sleep-wake.md) sleep, so
it is immune to any wall-clock adjustment.

## Source

```
  src/syscall/dispatch/router/surface_ops.rs   the six MkSurface*/MkDisplay* handlers, map_err
  src/kernel_core/surface_registry/vsync.rs      the phase-grid vblank and the serialization fix
```

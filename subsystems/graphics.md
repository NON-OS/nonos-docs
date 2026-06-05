# Graphics and Surfaces

A surface is a framebuffer a capsule owns, described to the kernel so it can be
shared with another capsule and presented to the display. The kernel does not
draw; it tracks surfaces, maps them between capsules, and flips them to the
display. The [architecture overview](../architecture/overview.md) covers this in
section 14; here is the lifecycle in full.

---

## The descriptor

A capsule allocates a framebuffer in its own address space and describes it to
the kernel (`src/kernel_core/surface_registry/types.rs`):

```
  SurfaceDescriptor
    width     u32    pixels
    height    u32    pixels
    stride    u32    bytes per scanline
    format    u32    FMT_ARGB8888 = 1
    byte_len  u64    total buffer size
    base_va   u64    the capsule's virtual base address for the buffer
    flags     u64
```

Note that `stride` is bytes per scanline, not pixels. A renderer that confuses the
byte stride with the pixel pitch produces diagonal scanline corruption, so the
descriptor names the unit explicitly.

## Handles

The kernel stores surfaces in a slot table (`SLOT_CAP = 256`). A handle encodes
the slot index and an epoch (`types.rs:70`):

```
  encode_handle(slot_idx, epoch) = (slot_idx as u64 << 32) | epoch
```

The epoch increments when a slot is reused. A handle to a freed surface therefore
carries a stale epoch and is rejected, rather than silently resolving to whatever
surface now occupies that slot. This is a use-after-free guard built into the
handle itself.

---

## Lifecycle

```
  producer capsule                 kernel surface registry        consumer capsule
    |  allocate framebuffer in own VA
    |  MkSurfaceRegister(desc) ------>  translate base_va pages to PA frames
    |                                   store in a slot, return surface id
    |                                   and a handle (slot << 32 | epoch)
    |  MkSurfaceShare(sid) ---------->  bump refcount, return a handle to share
    |                                                 ------------->  MkSurfaceAttach(handle)
    |                                                                   map the frames into
    |                                                                   the consumer VA, return
    |                                                                   the VA and a descriptor
    |  MkSurfacePresent(handle) ----->  route the framebuffer to the
    |                                   display backend and flip
    |  MkDisplayVsyncWait(0) -------->  block until the next vblank
```

### Register

`MkSurfaceRegister` (`src/syscall/dispatch/router/surface_handlers.rs:47`) walks
the capsule's framebuffer pages, translates each virtual page to its physical
frame, and records the frames in a slot. From then on the kernel works from those
recorded frames, not from the raw `base_va`, so presentation and attach do not
trust a pointer the capsule could change. It returns a surface id and a handle.
Registering requires `GraphicsSurfaceCreate`.

### Share and attach

`MkSurfaceShare` increments a refcount and returns a handle another capsule can
use. `MkSurfaceAttach` (`surface_handlers.rs:98`) maps the surface's backing
frames into the attaching capsule's page tables and returns the virtual address
where they now live, plus a copy of the descriptor. This is the one controlled way
two capsules share memory: a compositor attaches the surfaces its clients
registered and composites from them, without either side handing the other a raw
pointer. Attaching requires `GraphicsSurfaceMap`.

### Present

`MkSurfacePresent` (`surface_handlers.rs:127`) routes a surface's framebuffer to
the display backend, which flips the scanout to it. If no capsule owns the display
backend the kernel presents directly; otherwise the present is routed to the
graphics capsule. Presenting requires `GraphicsPresent`.

### Vsync

`MkDisplayVsyncWait` blocks until the next vertical blank and returns its deadline
(`src/kernel_core/surface_registry/vsync.rs`). The default cadence is 60 Hz
(`TARGET_HZ_DEFAULT = 60`):

```
  wait_for_vsync(display)
    period = 1 / 60 s
    deadline = last vblank + period, skipping any missed periods
    busy-wait until now >= deadline
    record deadline as the last vblank, return it
```

A compositor's frame loop calls present, then waits for vsync, so it paints at the
display's cadence rather than spinning. The wait requires
`GraphicsDisplayQuery`, the same capability as querying the display dimensions.

---

## Where rendering lives

The kernel's whole graphics role is the slot table, the VA-to-PA translation, the
attach mapping, the present flip, and the vsync clock. Everything visual, drawing
widgets, compositing windows, the cursor, the clock, the dock, runs in user-mode
capsules: the compositor, the window manager, the desktop shell, and the
applications. They register surfaces, share and attach them to compose, and
present at vsync. The kernel guarantees that a surface's backing memory is what was
registered, that a stale handle is caught, and that a flip reaches the display. It
makes no decision about what is drawn.

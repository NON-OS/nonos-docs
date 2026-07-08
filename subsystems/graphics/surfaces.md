# Surfaces

A surface is a framebuffer a capsule owns: a rectangle of ARGB pixels backed by physical frames,
registered with the kernel so it can be shared with a compositor and presented to the display. The
kernel does not draw; it tracks who owns which surface and mediates the frame sharing. This page
documents the surface itself and the registry. The code is `src/kernel_core/surface_registry/`.

## The descriptor and the slot

A capsule describes a surface with a `SurfaceDescriptor` (`types.rs:32`) and the registry keeps a
`Slot` (`table.rs:27`) for each live one:

```
  SurfaceDescriptor { width, height, stride, format, byte_len, base_va, flags }

  Slot { owner_pid, epoch, refcount, width, height, stride, format,
         flags, byte_len, owner_base_va, frames: Vec<PhysAddr> }
```

The only pixel format is `ARGB8888` (`PIXEL_BYTES = 4`), and the surface is backed by a vector of
physical frames the owner already had mapped. The slot records the owner pid, a refcount for
sharing, the geometry, and the frames; `owner_base_va` is the virtual address the owner registered
it at, kept so a self-attach can return it without remapping. Surfaces live in a fixed table of
`SLOT_CAP = 256` slots, and a surface is at most `MAX_PAGES_PER_SURFACE = 8192` pages, which is the
same framebuffer-sized ceiling the [DMA broker](../hardware-broker/dma.md) uses (one 4K ARGB
surface).

## Registration

`register_surface` (`table.rs:46`) validates the descriptor and claims a free slot:

```
  register_surface(owner_pid, desc, frames):
      reject if format != ARGB8888, or width/height == 0
      reject if stride < width * 4                    // stride must cover a row
      reject if frames empty or > MAX_PAGES_PER_SURFACE
      find a free slot, set refcount = 1
      return (sid, handle)
```

The validation is strict: the format must be the one supported format, the dimensions must be
non-zero, and the stride must be at least a full row of pixels, so a surface cannot be registered
with geometry that would let a later present read out of bounds. A full table returns `OutOfSlots`.
The call returns a surface id and a handle.

## The epoch-guarded handle

A surface handle packs the slot index and an epoch (`types.rs:70`):

```
  handle = (slot_index << 32) | epoch
```

The epoch is what makes a handle safe to hold across a slot being freed and reused. Every operation
that takes a handle, share, attach, present, decodes it and checks the epoch against the slot's
current epoch, rejecting a mismatch with `BadHandle`. So if a surface is released and its slot is
later reused for a different surface, a stale handle from the old surface does not silently address
the new one; it fails. `lookup_owned` additionally checks the caller is the owner (`NotOwner`
otherwise), so ownership operations cannot be performed on someone else's surface.

## Source

```
  src/kernel_core/surface_registry/types.rs   SurfaceDescriptor, the handle encoding, the caps
  src/kernel_core/surface_registry/table.rs    the slot table, register_surface, lookup_owned
```

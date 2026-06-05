# Memory

NØNOS is RAM-resident, and the memory subsystem is what makes that concrete: a
bitmap frame allocator over physical RAM, a paging manager that owns the page
tables, and a unified-VM step that moves the kernel into the upper half and tears
down the bootloader's identity map. The [architecture overview](../architecture/overview.md)
covers this in section 5; here is the full detail with sources.

---

## Physical memory

Physical memory is a bitmap frame allocator over 4 KiB frames
(`src/memory/phys`, `src/memory/frame_alloc`). It is seeded at boot from the EFI
memory map handed to the kernel (`src/kernel_core/init/memory.rs`), which marks
the usable regions. Allocation hands out single frames; a higher-level page
allocator (`src/memory/page_allocator`) carves named kernel virtual ranges out of
the address space, for example the per-pid kernel stacks.

```
  EFI memory map  ->  phys::init(start, end)  ->  bitmap of 4 KiB frames
                                                    allocate_frame / free_frame
```

## The heap

The kernel heap is a global allocator bootstrapped very early in core init
(`src/memory/heap/manager`, called at `core_init.rs:42`), before the full IDT is
loaded, so interrupt handlers and the rest of init can use `alloc`, `Vec`, and
`Box`. Everything above the frame allocator that needs dynamic memory goes
through it.

## Paging

Page tables are owned by a paging manager (`src/memory/paging/manager`). It
tracks address spaces, the active top-level table, and the mappings within each.
Each capsule gets its own address space with a private lower half and a shared
upper half; the manager is what installs and switches those.

---

## Unified VM

The pivotal step is `init_unified_vm` (`src/memory/unified/init/run.rs:35`),
called during microkernel init after the scheduler is up and before interrupt
routing. It takes the kernel from running on the bootloader's identity map to
running purely from the upper half.

The kernel half is the top quarter of the PML4
(`run.rs:27`: `KERNEL_HALF_START = 256`, `KERNEL_HALF_END = 512`):

```
  PML4 index    role
  ----------    ----
  256           direct map of physical memory
  511           kernel text
  256..511      the kernel half, shared into every capsule address space
  0..255        the low half, the bootloader's identity map (torn down)
```

The steps, in order:

```
  1. register the active PML4 (read from CR3) as the kernel address space
  2. confirm the bootloader populated the kernel half, PML4[256..511]
  3. probe the frame allocator: allocate one frame, then free it
  4. bring up the page allocator for kernel virtual ranges
  5. remap the LAPIC MMIO page into the upper half as uncached,
       and rebind the LAPIC base pointer to that virtual address
  6. tear down the low half, PML4[0..255], only when the kernel half
       holds at least two of its own mappings (run.rs:107)
```

Step 5 matters for correctness: the local APIC must be reachable from the upper
half before the low half is removed, and its mapping must be uncached so reads
and writes hit the device rather than a cache line. Step 6 is gated: the low half
is cleared only after the kernel half is confirmed populated, so the kernel never
removes the map it is executing from.

After this runs, the identity map the bootloader used to get into Rust is gone.
This is what RAM-resident means in practice. There is no lower-half scratch the
kernel can fall back on, so a stray lower-half pointer dereferenced in kernel
mode faults instead of silently reading boot leftovers. It also means the only
code mapped low is capsule code, each in its own address space, so a capsule
cannot reach kernel memory by walking a shared low half.

---

## Address space per capsule

```
  capsule address space

  upper half  (PML4[256..511])   kernel text, direct map, kernel stacks, LAPIC
                                 shared and identical in every capsule
  lower half  (PML4[0..255])     this capsule's code, data, user stack,
                                 mapped surfaces, and DMA buffers, private
```

A context switch into a capsule loads that capsule's PML4 (the
`switch_address_space` primitive on the [arch boundary](../architecture/overview.md)).
The upper half is the same physical tables in every space, so kernel mappings do
not need to be rebuilt per capsule; only the private lower half differs. The
canonical boundary at `0x0000_7FFF_FFFF_FFFF` separates the two, and the user
context builder refuses to place an entry point or stack above it
(`src/arch/x86_64/context/setup.rs:36`).

---

## Where surfaces and DMA fit

Two kinds of memory cross the capsule boundary in a controlled way. A surface is
a framebuffer a capsule owns; when it registers one, the kernel translates the
capsule's virtual pages to physical frames and records them, so presentation
works from the kernel's record rather than a raw pointer
([graphics](graphics.md)). A DMA grant pins a driver's buffer and translates it
to a physical address a device can reach ([hardware broker](hardware-broker.md)).
In both cases the kernel holds the physical truth and the capsule holds only a
mapping it was granted.

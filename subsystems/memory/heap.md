# The Kernel Heap

The kernel heap backs `alloc`, so `Vec`, `Box`, `BTreeMap`, and every other dynamic
structure in the kernel allocate through it. It comes up in two phases: a small
fixed buffer available before paging and the frame allocator exist, and then a
larger frame-backed region mapped at a fixed virtual base. Both phases drive the
same allocator, which zeroes memory on allocation and on free. The code is under
`src/memory/heap/`.

## The global allocator

The heap is the Rust global allocator (`src/memory/heap/manager/globals.rs:21`):

```
  #[global_allocator]
  static KERNEL_HEAP: SecureHeapAllocator = SecureHeapAllocator::new();
```

Because it is marked `#[global_allocator]`, every `alloc`-crate allocation in the
kernel routes to it, which is why bringing it up early in boot is what makes `Vec`
and friends available to the rest of initialisation. The allocator type is
`SecureHeapAllocator` (`src/memory/heap/types/`); the "secure" is the zeroing
policy described below.

## The two phases

The heap starts in bootstrap mode. A fixed static buffer is reserved in the kernel
image (`globals.rs:29`):

```
  static mut BOOTSTRAP_HEAP_MEMORY: BootstrapHeapMemory =
      BootstrapHeapMemory { data: [0u8; BOOTSTRAP_HEAP_SIZE] };
  static USING_BOOTSTRAP: AtomicBool = AtomicBool::new(true);
```

`USING_BOOTSTRAP` begins `true`, so the earliest allocations, made before the frame
allocator and paging are up, are served from this fixed buffer built into the
image. It is small and fixed, so it is only meant to carry init far enough to map
the real heap. Once that is done, `init` flips `USING_BOOTSTRAP` to `false`
(`init.rs:38`) and every subsequent allocation is served from the mapped region.

## Bringing up the real heap

`init` (`src/memory/heap/manager/init.rs:26`) allocates the backing frames, maps
them at the heap's virtual base, and hands the region to the allocator:

```
  init():
      if the heap is already initialised -> Ok
      heap_size  = layout::KHEAP_SIZE
      heap_pages = ceil(heap_size / PAGE_SIZE)
      frames     = allocate_heap_frames(heap_pages)     one frame per page
      heap_start = map_heap_memory(frames)              map at KHEAP_BASE, R|W
      KERNEL_HEAP.init(heap_start, heap_size)
      HEAP_STATS.set_total_size(heap_size)
      USING_BOOTSTRAP = false
```

`allocate_heap_frames` (`init.rs:42`) pulls one frame at a time from the
[frame allocator](physical-frames.md) and returns `HeapError::FrameAllocationFailed`
if any request comes back empty, so a heap that cannot be fully backed fails init
rather than coming up partially. `map_heap_memory` (`init.rs:53`) maps each frame in
turn at `KHEAP_BASE + i * PAGE_SIZE` with `READ | WRITE` permissions through the
[paging manager](paging-manager.md), returning `HeapError::MappingFailed` on any
failure. The heap is therefore a contiguous virtual range at a fixed base, backed by
frames that need not be contiguous in physical memory. `init` is idempotent: a
second call returns `Ok` immediately once the heap is initialised.

## Zeroing on allocation and free

Two flags govern the allocator's zeroing policy, and both default to on
(`globals.rs:24`):

```
  HEAP_ZERO_ON_ALLOC  AtomicBool = true
  HEAP_ZERO_ON_FREE   AtomicBool = true
```

With zero-on-alloc, memory handed out never carries the contents of a previous
allocation, so a bug that reads uninitialised memory sees zeros rather than stale
kernel data. With zero-on-free, memory is wiped as it is returned, so freed data
does not sit in the heap waiting to be read back through a later allocation. This is
the same defence-in-depth posture the [ZeroState](../../security/README.md) model
applies to capsule memory, applied here to the kernel's own heap, and it is what the
"secure" in `SecureHeapAllocator` names.

## Statistics and time

The heap keeps `HEAP_STATS` (`HeapStatistics`) with the total size set at init and
allocation counts maintained thereafter, and `get_timestamp` (`globals.rs:32`) reads
the TSC directly for timestamping, since the heap can be exercised before the higher
time bases are calibrated.

## Errors

The heap error type (`src/memory/heap/error/`) surfaces two failures from init:
`FrameAllocationFailed` when the frame allocator cannot back the whole heap, and
`MappingFailed` when a page cannot be mapped at the heap base.

## Where this connects

The heap depends on the [frame allocator](physical-frames.md) for its backing pages
and the [paging manager](paging-manager.md) to map them, so it is brought up in
`init_unified_vm`'s wake during [boot](../boot/README.md), after those two are live and before
the rest of init needs dynamic allocation. Its virtual base and size are
`layout::KHEAP_BASE` and `layout::KHEAP_SIZE` in the memory layout constants.

## Source

```
  src/memory/heap/manager/globals.rs  the global allocator, bootstrap buffer, flags
  src/memory/heap/manager/init.rs      init, allocate_heap_frames, map_heap_memory
  src/memory/heap/types/               SecureHeapAllocator and HeapStatistics
  src/memory/heap/error/               HeapError
  src/memory/layout/                    KHEAP_BASE and KHEAP_SIZE
```

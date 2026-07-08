# The Page Allocator

The [frame allocator](physical-frames.md) hands out raw physical frames, and the
[paging manager](paging-manager.md) maps them. The page allocator sits above both: it is the
kernel's tracked allocator for whole virtual page ranges, the layer that carves a range of
kernel virtual address space, backs it with frames, zeroes it, and remembers it so it can be
freed and zeroed again. Per-process kernel stacks and similar fixed kernel allocations come
from here. The code is under `src/memory/page_allocator/`.

## What it allocates

A request is a size in bytes; the allocator rounds it up to whole pages and returns a kernel
virtual address for the base of the range. `allocate_page` (`manager/alloc.rs:27`) is the
core:

```
  allocate_page(size):
      reject if not initialized, size == 0, or size > MAX_ALLOCATION_SIZE (1 GiB)
      reject if tracked pages >= MAX_TRACKED_PAGES (100_000)
      page_count = ceil(size / PAGE_SIZE)
      va = allocate_virtual_pages(page_count)      // backed by the buddy allocator
      pa = translate(va)                           // resolve the backing frame
      record AllocatedPage { page_id, va, pa, time, size }
      write_bytes(va, 0, total_size)               // zero the whole range
      return va
```

The virtual range and its frame backing come from the buddy allocator
(`crate::memory::buddy_alloc`, via `manager/mapping.rs:22`), which allocates the contiguous
virtual pages and maps them; the page allocator adds tracking, physical-address resolution,
and the zeroing. Every allocation is zeroed before it is returned, so a caller never sees a
previous tenant's bytes. The size is bounded above at one gigabyte and the number of live
tracked allocations at one hundred thousand, so neither a single oversize request nor an
unbounded number of small ones can run the tracking table away.

## What it tracks

Each live allocation is an `AllocatedPage` (`types/page.rs:20`) kept in a `Vec` behind the
allocator's mutex:

```
  struct AllocatedPage {
      page_id:       u64,        // monotonic, from INITIAL_PAGE_ID = 1
      virtual_addr:  VirtAddr,
      physical_addr: PhysAddr,
      allocation_time: u64,      // TSC at allocation
      size:          usize,      // rounded-up byte size
  }
```

The record is what lets the allocator answer `get_page_info`, `is_allocated`, and the free
path by virtual address; the monotonic `page_id` and the TSC timestamp make an allocation
identifiable in a dump. The allocator is a single global, `PAGE_ALLOCATOR`, a `Mutex` around
the `PageAllocator` struct (`manager/globals.rs:21`), initialized once at unified-VM bring-up.

## Freeing

`deallocate_page` (`manager/dealloc.rs:26`) reverses the allocation and zeroes on the way
out as well:

```
  deallocate_page(va):
      idx = tracked page with virtual_addr == va      else PageNotFound
      page = remove(idx)
      write_bytes(va, 0, page.size)                    // zero before unmap
      free_virtual_pages(va, page.size / PAGE_SIZE)    // buddy unmap + frame free
      record deallocation
```

Freeing an address that was never allocated here is `PageNotFound` rather than a silent
unmap, so the allocator will not tear down a range it did not carve. The range is zeroed
before it is unmapped, which pairs with the zero-on-allocate to give the property that page
memory holds no stale content either when handed out or after being reclaimed; this is the
per-allocation half of the [zeroization](zeroization.md) posture.

## Statistics

`AllocatorStats` (`types/stats.rs:19`) counts total allocations and deallocations, live page
count, total bytes, and a peak page high-water mark maintained with a compare-exchange loop
so a concurrent allocation cannot lose a peak update. The snapshot is available through
`get_stats`, `get_allocation_count`, `get_total_bytes_allocated`, and `get_peak_pages`
(`manager/api.rs`), which makes the kernel's virtual-range usage observable at runtime.

## Where it sits

```
  page_allocator   tracked virtual-range allocation, zero-on-alloc/free, stats
      |
      v
  buddy_alloc      contiguous virtual pages, frame backing, mapping and unmapping
      |
      v
  frame allocator  raw physical frames        paging manager  the mappings
```

The page allocator is the tracked, zeroing, size-bounded front the rest of the kernel calls
for whole-range allocations; the buddy allocator underneath owns the virtual-range bookkeeping
and the actual map and unmap, over the [frame allocator](physical-frames.md) and
[paging manager](paging-manager.md). The general-purpose byte allocator capsules and kernel
code reach through `alloc` is the separate [heap](heap.md); the page allocator is for
page-granular kernel ranges.

## Source

```
  src/memory/page_allocator/manager/alloc.rs    allocate_page, zero-on-alloc
  src/memory/page_allocator/manager/dealloc.rs  deallocate_page, zero-on-free
  src/memory/page_allocator/manager/mapping.rs  buddy-allocator backing and translation
  src/memory/page_allocator/manager/api.rs      the public surface and init
  src/memory/page_allocator/types/page.rs        AllocatedPage, PageInfo
  src/memory/page_allocator/types/stats.rs       AllocatorStats and the peak high-water mark
  src/memory/page_allocator/constants.rs         the size and tracking bounds
```

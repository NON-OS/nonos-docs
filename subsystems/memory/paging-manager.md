# The Paging Manager

The paging manager owns the kernel's view of virtual memory. It tracks every
address space, every mapping the kernel has installed, the page-table root that is
active on the CPU, and the allocation of address-space identifiers. It is a single
global behind a lock, and every path that maps or unmaps a page goes through it.
This page documents its state, the permission model it installs, the address-space
record, and the mapping interface with its typed helpers and their exact behaviour.
The code is under `src/memory/paging/`.

## The manager state

The whole tracked state of virtual memory is one structure
(`src/memory/paging/manager/core/types.rs:24`):

```
  PagingManager
    active_page_table  Option<PhysAddr>          the root last loaded into CR3
    active_asid        Option<u32>               the ASID active on the last
                                                 switch, for TLB shootdown scope
    mappings           BTreeMap<u64, PageMapping> every mapping, keyed by VA
    address_spaces     BTreeMap<u32, AddressSpace> every address space, by ASID
    next_asid          u32                       next ASID to hand out
    initialized        bool
```

`next_asid` starts at `FIRST_USER_ASID`, so user address spaces are numbered above
the kernel's reserved `KERNEL_ASID`. `active_asid` is `None` before any process
has been dispatched, when the kernel is still running on the boot page tables with
no user CR3 active; once a process is switched to, it records that process's ASID
so the TLB shootdown wrappers can scope their invalidations to the right address
space. The `mappings` and `address_spaces` maps are ordered `BTreeMap`s, so the
manager can answer range and lookup queries over what it has installed. The whole
structure lives behind a single lock at the module boundary, which makes the
interrupt discipline below necessary.

## Page permissions

Every mapping carries a `PagePermissions`, a `u32` bitfield with a named constant
per bit (`src/memory/paging/types/permissions/flags.rs:20`):

```
  READ  WRITE  EXECUTE  USER  GLOBAL
  NO_CACHE  WRITE_THROUGH  DEVICE
  COW  DEMAND  ZERO_FILL  SHARED  LOCKED
```

The operations on it are the usual set algebra as `const fn`s: `contains`,
`union`, `insert`, `remove`, and `empty`. The one predicate that is a security
invariant rather than a convenience is `is_wx_violation` (`flags.rs:60`):

```
  is_wx_violation(self) = self.contains(WRITE) and self.contains(EXECUTE)
```

A permission set is a write-execute violation exactly when it is both writable and
executable. No page in the system is ever supposed to be both, and this predicate
is the test that enforces it. The enforcement point and the guarantees around it
are documented in full on the [hardening](hardening.md) page; here it is enough to
know that the permission model can name a W^X violation and that the manager's
callers check for it. The remaining flags describe caching (`NO_CACHE`,
`WRITE_THROUGH`, `DEVICE`), sharing and lifecycle (`SHARED`, `LOCKED`, `COW`), and
lazy population (`DEMAND`, `ZERO_FILL`), the last two of which are the
[fault handler's](faults.md) concern.

## Address spaces

An address space is a small record tying an ASID to a page-table root and a
process (`src/memory/paging/types/address_space.rs:22`):

```
  AddressSpace
    asid           u32        the address-space identifier
    cr3_value      PhysAddr   the page-table root for this space
    process_id     u32        the owning process
    creation_time  u64
```

`is_kernel` (`address_space.rs:34`) reports whether the ASID is the reserved
`KERNEL_ASID`. The kernel's own address space is registered during unified-VM init
and shared, mapped identically, into the upper half of every process's space; each
process gets its own `AddressSpace` with a fresh ASID and its own `cr3_value` whose
lower half is private. The manager keeps these in `address_spaces` and consults
them when switching CR3 and when scoping TLB shootdowns.

## Mapping a page

The public entry point is `map_page` (`src/memory/paging/manager/api/mapping.rs:24`),
and the first thing it does is disable interrupts around the manager lock:

```
  map_page(va, pa, perms):
      without_interrupts(||
          PAGING_MANAGER.lock().map_page(va, pa, perms, Size4KiB, &PAGING_STATS))
```

The interrupt discipline is not incidental and the source explains it. The manager
is a `spin::Mutex`. If a timer interrupt fired on this CPU while the lock is held,
the preemption path in the ISR would call `switch_to_process_address_space`, which
takes the same lock, and the CPU would deadlock on its own mutex. Disabling
interrupts across the critical section closes that window. Every mapping and
unmapping entry point in this module follows the same pattern. Inside the lock the
manager walks the page-table levels, allocating intermediate tables from the
[frame allocator](physical-frames.md) as needed, installs the leaf entry, records
the `PageMapping` in its `mappings` map, and updates `PAGING_STATS`. `map_huge_page`
is the same with a caller-chosen `PageSize`.

## The typed helpers

Most callers do not build a raw permission set; they call a helper that encodes the
correct permissions and cache mode for a kind of memory, and these helpers are
where the system's memory policy is written down (`api/mapping.rs`).

```
  map_kernel_page      READ | WRITE | GLOBAL
  map_user_page(w)     READ | USER (+ WRITE if w)
  map_device_memory    READ | WRITE | NO_CACHE | DEVICE
  map_user_mmio        USER | READ | WRITE | NO_CACHE | DEVICE
  map_user_dma         USER | READ | WRITE                  (write-back cacheable)
```

Three details in these are worth stating because they are correctness, not style.
None of them set `EXECUTE`, so device, MMIO, and DMA mappings are non-executable by
construction. The MMIO helpers set `NO_CACHE` because device registers must not be
cached, while the DMA helper deliberately does not: on x86_64 PCI devices snoop the
cache, so a coherent DMA buffer is write-back cacheable, and marking it uncached or
write-combining would be wrong. And `map_user_mmio` and `map_user_dma` roll back on
partial failure: if the `n`-th page of a range fails to map, the helper unmaps the
`n-1` pages it already installed before returning the error, so a failed mapping
never leaves a partial range behind.

The source also records who is allowed to call the user device helpers: the comment
on `map_user_mmio` states that the caller is the hardware broker and no other path
is permitted to expose physical memory to a capsule, and that the helper does not
itself consult the broker tables, so the broker is responsible for confirming the
physical range belongs to a BAR the calling process claimed. The
[hardware broker](../hardware-broker/README.md) page covers that check.

## Unmapping and TLB shootdown

`unmap_page` (`api/mapping.rs:62`) unmaps under the same interrupt discipline,
returns the physical address that was mapped along with the permissions and size,
and records the unmapping in the stats. `unmap_range` walks 4 KiB pages across a
byte length rounded up and unmaps each, stopping at the first failure and returning
it. The unmap path is also where cross-CPU TLB coherency happens: as the comments
on `unmap_user_mmio` and `unmap_user_dma` note, `unmap_page` emits a per-ASID SMP
TLB shootdown through the manager's shootdown wrappers
(`src/memory/paging/manager/shootdown.rs`), scoped by the `active_asid` recorded in
the manager state, so a mapping removed on one CPU is invalidated on the others
that share the address space rather than lingering in their TLBs.

## Where this connects

The manager is brought up, and the kernel's own mappings established, by the
[unified-VM init](unified-vm.md), which also tears down the bootloader's low-half
identity map once the kernel half is confirmed. The `DEMAND`, `ZERO_FILL`, and
`COW` permission bits are populated lazily by the [fault handler](faults.md). The
W^X invariant named here is enforced and its guarantees stated on the
[hardening](hardening.md) page. And every intermediate page table this manager
allocates comes from the [physical frame allocator](physical-frames.md).

## Source

```
  src/memory/paging/manager/core/types.rs        the PagingManager state
  src/memory/paging/types/permissions/flags.rs   PagePermissions and is_wx_violation
  src/memory/paging/types/address_space.rs       the AddressSpace record
  src/memory/paging/manager/api/mapping.rs        map_page and the typed helpers
  src/memory/paging/manager/shootdown.rs          the per-ASID TLB shootdown
```

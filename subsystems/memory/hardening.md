# Memory Hardening

Beyond mapping memory correctly, the memory subsystem enforces a set of safety
invariants: no page is ever both writable and executable, guard pages catch access
that runs off the end of a region, stack canaries catch overflow, and allocation
tracking catches double-free and use-after-free. The write-execute rule is enforced
on the mapping path itself; the rest is managed by a single hardening manager with a
running tally of every violation it has seen. This page documents each, and in
particular it is where the W^X enforcement point named on the
[paging manager](paging-manager.md) page is actually located. The code is under
`src/memory/hardening/`, and the enforcement gate is in the paging manager.

## Write-execute exclusion, and where it is enforced

The invariant is that no page is both writable and executable. It is enforced at the
moment a mapping is installed, in `map_page_in_asid`
(`src/memory/paging/manager/mapping/map_in_asid.rs:37`):

```
  map_page_in_asid(asid, va, pa, permissions, size, stats):
      if not initialized                    -> NotInitialized
      if permissions.is_wx_violation()      -> WXViolation
      pte_flags = permissions.to_pte_flags()
      install the mapping, record the stat
```

The check runs before the page-table entry is computed, so a permission set that is
both `WRITE` and `EXECUTE` is rejected with `PagingError::WXViolation` and no PTE is
ever written for it. This is the real gate: the paging manager will not install a W+X
page into any address space, so the invariant holds by construction of every mapping
rather than by a scan after the fact. The permission-model side of it, the
`is_wx_violation` predicate, is on the paging page; this is the enforcement.

The hardening manager carries a second form of the same check for callers that reason
in terms of writable and executable booleans rather than a permission set,
`validate_wx_permissions` (`src/memory/hardening/manager/validation.rs:18`):

```
  validate_wx_permissions(addr, writable, executable):
      if writable and executable:
          increment the W^X violation counter
          return Err("W^X violation: memory cannot be both writable and executable")
```

It is exposed as `validate_memory_permissions` (`hardening/manager/api.rs:20`), and a
rejection here also bumps the `wx_violations` statistic so the manager can report how
many attempts it has refused.

## Guard pages

A guard page is an address the manager marks so that any access to it is a detected
violation rather than a silent read or write. They are added and removed through the
API (`hardening/manager/api.rs:50`):

```
  add_guard_page(addr, guard_type)     insert a GuardPage of one page at addr
  remove_guard_page(addr)              remove it, error if not present
  check_guard_page_access(addr)        true if addr is a guard page (+ counter)
```

`add_guard_page` records a `GuardPage { addr, size: PAGE_SIZE, protection_type }` in a
map keyed by address. `check_guard_page_access` (`api.rs:28`) tests whether a faulting
address is in that map, and if it is, increments the guard-violation counter and
returns true, so the fault path can treat a guard-page hit as a deliberate boundary
being crossed rather than an ordinary fault. Guard pages are how the manager turns the
region just past a stack or a buffer into a tripwire.

## Stack canaries

Stack overflow is caught with canaries (`hardening/manager/api.rs:78`):

```
  setup_stack_canary(stack_base, stack_size):
      value = generate_stack_canary()
      record StackCanary { value, stack_base, stack_size }
      write value volatile at stack_base + stack_size - 8
      return value
  check_stack_canary(stack_base)   verify the canary is intact
  clear_stack_canary(stack_base)   remove it
```

`setup_stack_canary` generates a canary value, records it, and writes it with a
volatile store eight bytes below the top of the stack, the location a write that ran
off the end of the stack would overwrite first. `check_stack_canary`
(`check_stack_integrity`) reads it back and fails if it has changed, which is the
signal that the stack overflowed into its guard word. The volatile write ensures the
compiler cannot elide the canary store as dead.

## Allocation tracking

The manager tracks allocations to catch lifetime errors (`api.rs:37`):

```
  track_allocation(addr, size)     record a live allocation
  track_deallocation(addr)         retire it, detecting a double free
  validate_heap_integrity(addr, size)   detect heap corruption over a range
```

`track_allocation` records an allocation and `track_deallocation` retires it; a
deallocation of an address that is not currently tracked is a double free, and an
access to a retired address is a use-after-free, both of which the tracker can detect
and count. `validate_heap_integrity` runs `detect_heap_corruption` over a range to find
metadata that has been overwritten.

## The violation tally

Every check above feeds one running snapshot (`api.rs:64`):

```
  HardeningStatsSnapshot
    guard_violations  wx_violations  stack_overflows  heap_corruptions
    double_frees      use_after_free
    total_guard_pages  active_canaries  tracked_allocations
```

`get_hardening_stats` returns it, so the number of W^X mappings refused, guard pages
hit, canaries broken, and lifetime errors caught is observable at runtime, alongside
the count of guard pages, canaries, and allocations currently tracked.

## Where this connects

The W^X gate sits on the [paging manager](paging-manager.md)'s install path, so it
applies to every mapping the manager makes, including the ones the
[fault handlers](faults.md) install. The guard pages and canaries protect the
per-process kernel stacks the [page allocator](page-allocator.md) carves and the
[heap](heap.md). The W^X invariant here is also one of the properties the
[verification stack](../../../verification/README.md) proves at the encoding level,
in the kernel isolation proofs.

## Source

```
  src/memory/paging/manager/mapping/map_in_asid.rs  the W^X enforcement gate
  src/memory/hardening/manager/validation.rs         validate_wx_permissions, guard test
  src/memory/hardening/manager/api.rs                the hardening surface
  src/memory/hardening/types/                         GuardPage, StackCanary, snapshots
  src/memory/hardening/stats/                         the violation counters
```

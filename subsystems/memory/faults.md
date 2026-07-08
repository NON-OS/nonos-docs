# Page Faults: Demand Paging and Copy-on-Write

The paging manager turns two kinds of page fault into deferred work. A fault on a
page that is not present backs it on demand, allocating and mapping a frame the
first time it is touched. A write fault on a page that is present makes a private
copy, the copy-on-write path. Every other fault is unhandled, which means the fault
path treats it as an error and kills the offending capsule or traps a real kernel
bug. This page documents the dispatch and both handlers. The code is under
`src/memory/paging/manager/faults/`.

## The dispatch

`handle_page_fault` (`faults/handler.rs:25`) reads the x86 page-fault error code and
routes on two of its bits:

```
  handle_page_fault(va, error_code, stats):
      record the fault
      if error_code has PF_WRITE and PF_PRESENT   -> copy-on-write
      if error_code has no PF_PRESENT             -> demand paging
      otherwise                                    -> UnhandledPageFault
```

A write to a page that is present but not writable is a copy-on-write fault. A fault
on a page that is not present is a demand fault. Anything else, for example a
protection fault that is neither of those, is returned as `UnhandledPageFault`, and
the exception path that called in decides what to do with it. Each branch also
records its own statistic before dispatching.

## Demand paging

`handle_demand_fault` (`faults/demand.rs:27`) backs a not-present user page, and its
first two checks are guards, not allocation:

```
  handle_demand_fault(va, stats):
      if va is not in user space          -> UnhandledPageFault
      pid = current process
      if not demand_cap::charge(pid)       -> UnhandledPageFault
      frame = allocate_frame()             -> FrameAllocationFailed if none
      zero the frame through the direct map
      map va -> frame as READ | WRITE | USER, 4 KiB
```

The user-space check is a security boundary and the source is explicit about it: a
not-present fault in the kernel half is never a legitimate lazy mapping, and backing
one silently would hand a capsule memory in the kernel's address range. So a
kernel-half demand fault is surfaced as unhandled, which kills the faulting user or
traps the real kernel bug rather than papering over it. Only user-space addresses
are ever demand-backed.

The budget check is a denial-of-service boundary. Each demand-backed page is charged
against the faulting process's demand budget through `demand_cap::charge`
(`faults/demand_cap.rs`), and a process that has exhausted its budget is refused
here and killed by the fault path, so a runaway capsule cannot fault-in pages until
it exhausts physical memory. Only after both guards pass does the handler allocate a
frame, zero it through the direct map so it cannot carry the contents of a previous
allocation, and map it into the faulting address writable and user-accessible.

## Copy-on-write

`handle_cow_fault` (`faults/cow.rs:27`) handles a write to a present, non-writable
page by giving the writer its own copy:

```
  handle_cow_fault(va, stats):
      new_frame = allocate_frame()          -> FrameAllocationFailed if none
      if translate_address(va) succeeds:
          copy the original page into new_frame through the direct map
      map va -> new_frame as READ | WRITE | USER, 4 KiB
```

It allocates a fresh frame, translates the faulting virtual address to the physical
frame it currently maps, copies that frame's contents into the new one through the
direct map, and remaps the faulting address to the new frame as writable and
private. After the fault the writer has its own copy and its write proceeds; any
other mapping of the original frame is untouched. This is what lets a page be shared
read-only between address spaces and split into private copies only when one side
writes.

## The direct map

Both handlers reach a physical frame's bytes through the direct map: the frame at
physical address `p` is addressable at `DIRECTMAP_BASE + p`. Demand paging uses it to
zero a fresh frame, and copy-on-write uses it to read the original frame and write
the copy. The direct map is one of the two kernel-half mappings the
[unified-VM init](unified-vm.md) confirms before it tears down the bootloader's low
half, which is why the fault handlers can rely on it.

## What is not handled, and why that is safe

The handler deliberately does little. A kernel-half not-present fault, a present
protection fault that is not a write, and anything that is neither demand nor
copy-on-write all return `UnhandledPageFault`. Returning an error rather than
guessing is the safe default: the exception path that invoked the handler kills a
faulting capsule or halts on a genuine kernel fault, instead of the handler silently
backing memory it should not. The handlers only ever create user-space,
budget-charged, zeroed or copied mappings, and never touch the kernel half.

## Where this connects

The fault handler is invoked from the page-fault exception vector in the
[interrupt layer](../interrupts/README.md), which passes it the faulting address from `CR2`
and the error code. The frames it allocates come from the
[physical frame allocator](physical-frames.md), and the mappings it installs go
through the [paging manager](paging-manager.md)'s `map_page` under the same interrupt
discipline.

## Source

```
  src/memory/paging/manager/faults/handler.rs   the dispatch on the error code
  src/memory/paging/manager/faults/demand.rs     demand paging and its guards
  src/memory/paging/manager/faults/demand_cap.rs the per-process demand budget
  src/memory/paging/manager/faults/cow.rs        copy-on-write
```

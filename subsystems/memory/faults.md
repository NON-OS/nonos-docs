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

## Security analysis

The fault path is where a page fault, an event a hostile capsule can generate at will, is turned
into either a bounded, safe mapping or a kill. Three properties keep that conversion tight.

**Demand paging is user-space only.** `handle_demand_fault` (`faults/demand.rs:37`) rejects any
not-present fault whose address is not in user space (`layout::in_user_space`) and returns
`UnhandledPageFault`. This is the boundary that matters most: a not-present fault in the kernel
half is never a legitimate lazy mapping, and backing one silently would hand a capsule a page in
the kernel's own address range. So the only addresses the handler will ever demand-back are
user-space ones, and a kernel-half fault is surfaced as unhandled, which traps the real kernel bug
instead of papering over it.

**Every backed page is zeroed before it is mapped.** Both handlers reach the fresh frame through
the direct map and scrub it before it enters the address space: demand paging writes 4 KiB of zeros
(`faults/demand.rs`), and copy-on-write copies the original page's bytes into the new frame
(`faults/cow.rs`). A lazily-backed page therefore never carries the previous tenant of that frame,
which is the per-fault half of the [zeroization](zeroization.md) posture. The frame itself was also
zeroed on free, so this is defence in depth rather than the only scrub.

**The demand budget is a denial-of-service bound.** Each backed page is charged against the faulting
process through `demand_cap::charge` (`faults/demand_cap.rs:49`), which caps a single process at
`MAX_DEMAND_PAGES` (4096 pages, 16 MiB) (`demand_cap.rs:27`). Stacks, heap, and code are mapped eagerly,
so a healthy capsule demand-faults essentially nothing; a process over budget is refused with
`UnhandledPageFault` and killed by the exception path, so a runaway capsule cannot fault-in pages until
it drains physical memory. The saturation path is metered, not open: the counter table holds
`MAX_TRACKED` (128) processes, and when it fills with live faulters an untracked process is charged
against a shared `overflow` budget bounded by the same `MAX_DEMAND_PAGES` and then refused
(`demand_cap.rs:83`), so there is no unmetered path around the cap. The overflow resets when a tracked
slot frees, since the pressure that forced it is gone (`demand_cap.rs:75`). Pid 0 (no current process)
is charged as always-allowed (`demand_cap.rs:50`), which is the kernel's own faults, not a capsule's.

## Debugging page faults

Every page fault narrates: the exception path
(`src/interrupts/handlers/exceptions/page_fault.rs:31`) calls `dump_trap` unconditionally at
`page_fault.rs:35`, before it even tries to handle the fault at `page_fault.rs:44`, so a handled
demand or copy-on-write fault prints a `[TRAP PF]` line just like an unhandled one. What distinguishes
a handled fault is that nothing follows the line: no `Segmentation fault`, no `KERNEL PANIC`. This is
tolerable only because demand faults are rare by design (eager mapping, see the demand budget above); a
flood of `[TRAP PF]` lines is itself the signal that a capsule is faulting when it should not. The fixed
shape is:

```
  PF <trap dump>                                   dump_trap: the frame, error code, and CR2
  Guard page violation detected                    the address is a registered guard page
  Segmentation fault: user process accessed ...    a user fault the handler could not back (kill)
  KERNEL PANIC: Page fault at address ...          a kernel-half fault (halt)
    Attempted to execute from non-executable page  instruction-fetch fault (NX / W^X hit)
    Attempted to write to read-only page           write to a read-only mapping
    Attempted to read from non-present page        read of an unmapped address
```

The `[DEMAND-CAP] per-process page budget hit, killing pid=<hex>` line comes from
`demand_cap.rs:63` and is printed exactly once as a process crosses its 16 MiB budget, so a capsule that
dies right after a burst of allocation is a budget kill, not a mapping bug. The companion
`[DEMAND-CAP] tracking table saturated, overflow budget spent, killing pid=<hex>` line
(`demand_cap.rs:89`) is the rarer case: the 128-slot counter table was full of live faulters and the
shared overflow budget was spent, so this pid was refused through the saturation path rather than its own
per-process counter. The three "Attempted to ..."
lines are chosen off the error-code bits (`is_instruction_fetch`, `is_write`), so a kernel panic
tells you which access the kernel made: an execute line on a kernel panic is a W^X or NX violation
inside the kernel, a write line is a write to a read-only kernel mapping. A user "Segmentation fault"
line means the handler saw a fault it will not back (a kernel-half address from ring 3, a protection
fault that is neither demand nor copy-on-write), and the process exits with `-11` (SIGSEGV). A
"Guard page violation detected" means the faulting address was registered as a guard page on the
[hardening](hardening.md) side, so it is a deliberate tripwire, an overrun off a stack or buffer,
rather than an ordinary miss.

## Source map

```
  src/memory/paging/manager/faults/handler.rs     the dispatch on the error code
  src/memory/paging/manager/faults/demand.rs       demand paging and its guards
  src/memory/paging/manager/faults/demand_cap.rs   the per-process demand budget
  src/memory/paging/manager/faults/cow.rs          copy-on-write
  src/interrupts/handlers/exceptions/page_fault.rs the exception vector and its markers
```

Every reference above is verified against those trees. The error code and CR2 come from the
[interrupt layer](../interrupts/README.md), the frames come from the
[physical frame allocator](physical-frames.md), the mappings go through the
[paging manager](paging-manager.md), and the zero-on-back pairs with the
[zeroization](zeroization.md) posture.

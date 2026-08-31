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

## The CPU protection bits

The software checks above are backed by hardware enforcement, and there is now a single owner of the
control registers. `init_module_memory_protection`
(`hardening/manager/verify/protection.rs:28`) no longer writes CR0/CR4 itself; it delegates to
`mmu::init_mmu()` (`protection.rs:30`). The source records why: this path used to write the registers in
parallel with `memory::mmu`, so two pieces of code owned the same bits and neither read back what stuck.
There is one owner now.

The real work is `MMU::initialize` (`mmu/mmu/init.rs:25`) calling `protect::apply`
(`mmu/mmu/protect/apply.rs:26`), which probes the part's features and turns on what it offers:

```
  apply():                                        # mmu/mmu/protect/apply.rs:26
      have = cpuid::supported()                   # actually probes CPUID
      if not have.nx: return Err(NxNotSupported)  # NX has no fallback
      live = cr4::enable(have)                     # SMEP/SMAP/UMIP, gated on CPUID
      ProtectionFlags {
          smep_enabled: live.smep, smap_enabled: live.smap, umip_enabled: live.umip,
          nx_enabled: efer::enable_nx(), wp_enabled: cr0::enable(),   # every field a read-back
      }
```

`cr0::enable` sets `CR0.WP`, so a read-only page is read-only even to ring 0: the kernel cannot
accidentally write through a mapping it marked read-only. `CR4.SMEP` stops the kernel from fetching
instructions out of any user page, and `CR4.SMAP` stops it from reading or writing user pages except
through an explicit `stac`/`clac` window, which is why the [usercopy](usercopy.md) boundary reaches user
bytes through the direct map rather than dereferencing the user pointer. `efer::enable_nx` turns on
execute-never, and unlike the others it is mandatory: `apply` returns `NxNotSupported` and refuses to
proceed if the part lacks it, because the directmap through which every user page is reached is built
with NX set and a part that ignored it would leave that whole window executable. Every field of the
returned `ProtectionFlags` is a read-back of what the hardware accepted, not what was requested, so
`get_protection_flags` answers with the machine's truth (a part that silently dropped SMAP reads back
`false`). `initialize` is idempotent (`init.rs:26`), so re-entry touches no register. On aarch64 and
riscv64 the same two guarantees come from PAN and the PXN/UXN table bits (`apply.rs:51`).

## Security analysis

Hardening is defence in depth: the mapping path and the allocators are already correct, and this
subsystem adds invariants that turn a bug that slips past them into a caught, counted event rather
than a silent corruption. Four properties carry that.

**W^X holds by construction.** The real gate is `map_page_in_asid` (`mapping/map_in_asid.rs:37`),
which rejects any `permissions.is_wx_violation()` set with `PagingError::WXViolation` before it
computes a page-table entry. No page in any address space is ever both writable and executable,
because no such PTE is ever written, not because a scan catches it after the fact. The
`validate_wx_permissions` form (`validation.rs:18`) is the same rule for callers reasoning in
booleans, and both feed the `wx_violations` counter. Combined with SMEP and NX on the device and DMA
helpers, a writable page is never a code-injection target.

**Kernel writes respect read-only, kernel fetches respect user.** `CR0.WP`, `CR4.SMEP`, `CR4.SMAP`,
`CR4.UMIP`, and NX in EFER move these guarantees from convention into the hardware. Without WP the kernel
could write through its own read-only mappings; without SMEP it could be tricked into executing a user
page; without SMAP it could dereference an unvalidated user pointer. `apply` (`mmu/mmu/protect/apply.rs:26`)
does branch on a real CPUID probe (`cpuid::supported()`, `apply.rs:29`) and enables SMEP/SMAP/UMIP only
where the part reports them, then records the read-back so a caller gating on `smap_enabled` sees the
truth on a part that dropped it. NX is the one property with no fallback: `apply` returns
`NxNotSupported` and refuses to bring the MMU up at all rather than run with an executable directmap
(`apply.rs:33`). The honest boundary is now SMEP/SMAP specifically: on a part that lacks them those flags
read back `false` and the software W^X and usercopy checks are the only line left, but the kernel knows
that from the read-back rather than assuming the write took.

**Guard pages and canaries are overrun tripwires.** A guard page (`api.rs:50`) marks an address so
that a fault on it is reported as a deliberate boundary crossing (`check_guard_page_access`,
`api.rs:28`) rather than an ordinary miss, and the page-fault path checks it first
(`page_fault.rs:63`). A stack canary (`api.rs:78`) is a value written with a volatile store eight
bytes below the top of a stack, the first thing an overflow overwrites, and `check_stack_canary`
fails if it changed. Both catch an off-the-end write that the mapping permissions alone would not.

**Lifetime errors are detectable and counted.** `track_allocation` / `track_deallocation`
(`api.rs:37`) let the manager name a double free (deallocating an untracked address) and a
use-after-free (touching a retired one), and the frame allocator has its own hard `DoubleFree` check
underneath. The honest limit is stronger than "opt-in" and worth stating plainly: as the tree stands,
nothing outside the hardening module calls `add_guard_page`, `setup_stack_canary`, or `track_allocation`.
The manager's guard-page map, canary table, and allocation tracker are implemented and their counters are
wired to the snapshot, but no allocator or stack-carving path currently registers anything with them, so
in practice these three tallies stay at zero. The one canary that does run is the sanitization module's
own stack canary (`memory_sanitization/api.rs:117`), which is separate from this manager's table. The W^X
gate and the CPU bits, by contrast, are unconditional and always in force. Closing this gap means having
the page allocator and heap register their regions as they carve and free them.

## Debugging hardening

The hardening subsystem is designed to make a violation visible rather than fatal-and-silent, so the
first tool is the running tally. `get_hardening_stats` returns the `HardeningStatsSnapshot`
(`api.rs:64`) with one counter per class:

```
  guard_violations   a guard page was hit          (add_guard_page had marked it)
  wx_violations      a W+X mapping was refused      (map_in_asid or validate_wx_permissions)
  stack_overflows    a canary came back changed
  heap_corruptions   detect_heap_corruption fired over a tracked range
  double_frees       a deallocation of an untracked address
  use_after_free     an access to a retired address
  total_guard_pages  active_canaries  tracked_allocations   what is currently live
```

A non-zero `wx_violations` with the system still running means the gate did its job: a caller tried
to map a W+X page and was refused with `WXViolation`, no PTE was written, and the count is the
evidence. On the fault side, the exact strings come from the page-fault handler, not this module:
`Guard page violation detected` (`page_fault.rs:64`) is printed when a faulting address matches a
registered guard page, and the "Attempted to execute from non-executable page" line on a
`KERNEL PANIC` is an NX or W^X hit reaching hardware. So a guard-page overrun shows up twice, as a
console line at the fault and as an increment of `guard_violations`, and reading them together
distinguishes a deliberate tripwire from an ordinary miss. The W^X refusal string itself,
`"W^X violation: memory cannot be both writable and executable"` (`validation.rs:26`), is what a
caller sees when it asks for a writable-executable mapping.

## Where this connects

The W^X gate sits on the [paging manager](paging-manager.md)'s install path, so it
applies to every mapping the manager makes, including the ones the
[fault handlers](faults.md) install, and the guard-page check is consulted first by the
[page-fault handler](faults.md). The guard pages and canaries protect the
per-process kernel stacks the [page allocator](page-allocator.md) carves and the
[heap](heap.md). The SMAP bit set here is what forces the [usercopy](usercopy.md)
boundary to go through the direct map. The W^X invariant is also one of the properties the
[verification stack](../../architecture/verification.md) proves at the encoding level,
in the kernel isolation proofs.

## Source map

```
  src/memory/paging/manager/mapping/map_in_asid.rs    the W^X enforcement gate
  src/memory/hardening/manager/validation.rs           validate_wx_permissions, guard test
  src/memory/hardening/manager/api.rs                  the hardening surface
  src/memory/hardening/manager/verify/protection.rs    delegates to mmu::init_mmu
  src/memory/mmu/mmu/init.rs                           MMU::initialize, idempotent bring-up
  src/memory/mmu/mmu/protect/apply.rs                  CPUID probe, CR0.WP/SMEP/SMAP/UMIP/NX, read-back
  src/memory/paging/tlb/write_protect.rs               enable_write_protection (CR0.WP)
  src/memory/hardening/types/                          GuardPage, StackCanary, snapshots
  src/memory/hardening/stats/record.rs                 the violation counters
```

Every reference above is verified against those trees. The W^X predicate is on the
[paging manager](paging-manager.md) page, the guard-page consumer is the [page-fault handler](faults.md),
and the SMAP-gated copy boundary is on the [usercopy](usercopy.md) page.

# mmap and munmap

A capsule grows its own address space with `MkMmap` and shrinks it with `MkMunmap`. These
are the only syscalls that let a capsule add anonymous, zeroed, user pages to its own space,
and they are deliberately narrow: they map private anonymous memory and nothing else, with
per-page frame allocation, W^X enforced by the mapping gate, and a rollback that leaves no
partial range behind. This page documents the map path, the unmap path, the per-process VA
allocator, and the per-process versus system accounting. The code is under
`src/syscall/microkernel/memory/`.

## Mapping: sys_mmap

`sys_mmap` (`src/syscall/microkernel/memory/mmap.rs:25`) takes `(addr, length, prot, flags)`
and returns the base virtual address it mapped, or a negative errno:

```
  sys_mmap(addr, length, prot, flags):                      # mmap.rs:25
      if length == 0 or length > MAX_MMAP_SIZE:  EINVAL      # 1 GiB cap, consts.rs:19
      if addr != 0 and not is_user_space(addr, length):  EPERM
      if addr != 0 and addr not page-aligned:  EINVAL
      perms = READ | USER (+WRITE if PROT_WRITE, +EXECUTE if PROT_EXEC)
      base = addr==0 ? reserve_va(pages) : addr             # allocator-owned vs fixed
      if fixed: refuse any page already mapped -> EINVAL      # do not orphan a live frame
      for each page:
          frame = allocate_frame() else rollback+NOMEM
          map_page(va, frame, perms) else free+rollback+NOMEM
          zero_frame(frame)
      record_mmap(pid, length, base)
      return base
```

Three properties are worth calling out. First, the size and range checks: `length` is capped
at `MAX_MMAP_SIZE` (1 GiB, `consts.rs:19`) and a fixed address must lie entirely within user
space, `is_user_space` (`consts.rs:25`) requiring `addr <= 0x0000_7FFF_FFFF_FFFF` with no
`addr + len` overflow, so a capsule cannot drive the map path over kernel addresses. Second,
W^X is not checked here but downstream: the handler passes `PROT_WRITE`/`PROT_EXEC` straight
into `PagePermissions` (`mmap.rs:39`), and a request for both is rejected by `map_page`'s
`WXViolation` gate (see [hardening](hardening.md)), so a writable-executable mmap fails at the
mapping, not at the syscall. Third, the fixed-address case refuses to map over an
already-present page (`is_mapped` loop, `mmap.rs:58`): overwriting a live PTE would orphan the
previous frame and corrupt the caller's own space, so a `MAP_FIXED`-style overlap is `EINVAL`
rather than a silent clobber. Allocator-chosen ranges are always fresh and skip that check.

Every page is allocated from the [frame allocator](physical-frames.md), mapped, and then
zeroed through `zero_frame` (`mmap.rs:87`), so a freshly mapped page never carries the previous
tenant of its frame. The map is a transaction: a frame-allocation or mapping failure part way
through rolls back the pages already installed (`rollback_mapped_pages`) and releases the
reserved VA (`release_va`) before returning `ENOMEM` (`mmap.rs:70`, `:82`), so a failed mmap
leaves the address space exactly as it found it.

## The per-process VA allocator

An `addr == 0` request is allocator-owned: the base comes from `reserve_va`
(`src/syscall/microkernel/memory/va/reserve_va.rs:19`), which reserves a run of pages from the
calling process's own `MmapVa` allocator held in its PCB
(`with_process_mut(pid, |pcb| pcb.mmap_va.lock().reserve(pages))`). The allocator is per
process, so two capsules mapping at the same time draw from disjoint spaces and one cannot
reserve into another's range. `release_va` returns a run to that same per-process allocator on
unmap or on a mid-map rollback. This is distinct from the eager `memory.next_va` the PCB also
carries; the mmap path uses the dedicated `mmap_va` allocator.

## Unmapping: sys_munmap

`sys_munmap` (`src/syscall/microkernel/memory/munmap.rs:26`) takes `(addr, length)` and
reverses the map:

```
  sys_munmap(addr, length):                                 # munmap.rs:26
      if addr == 0 or length == 0:  EINVAL
      if addr not page-aligned:  EINVAL
      if length > MAX_MMAP_SIZE or not is_user_space(addr, length):  EINVAL
      for each page:
          if unmap_page(va) -> Ok(phys):  deallocate_frame(phys)
      release_va(addr, pages)
      record_munmap(pid, length, addr)
      return 0
```

Each page is unmapped through `unmap_page` (`munmap.rs:42`), which removes the PTE and emits
the per-ASID TLB shootdown documented on the [paging manager](paging-manager.md) page, and the
returned physical frame is handed back to the frame allocator with `deallocate_frame`
(`munmap.rs:43`). The scrub is not a separate step here: `deallocate_frame` zeroes the frame on
return (the free-path arm of the [zeroization](zeroization.md) posture), so an unmapped page's
contents do not survive into the next allocation. The range is bounded to user space with the
same `is_user_space` check as the map path (`munmap.rs:36`), so an out-of-range or oversized
request cannot drive `unmap_page`/`deallocate_frame` over kernel addresses or wrap the
`addr + i * PAGE_SIZE` arithmetic. Finally the VA run is released back to the per-process
allocator (`munmap.rs:48`).

## Per-process versus system accounting

`record_mmap` and `record_munmap` update two tallies held in
`src/syscall/microkernel/memory/accounting/state.rs`:

- Per-process bytes: `OWNERS`, a fixed `[Owner { pid, bytes }; 64]` table (`state.rs:41`),
  tracks how many bytes each pid currently holds mapped. It is a fixed array rather than a map
  because the update runs under a lock on the mmap hot path.
- System bytes: `SYSTEM_BYTES`, one `AtomicU64` (`state.rs:44`), is the total across all
  processes, so the whole-system mmap footprint is one atomic read.
- A recent-event ring: `EVENTS`, a fixed `[Event; 20]` with `EVENT_CURSOR` (`state.rs:42`),
  records the last 20 map/unmap events (pid, kind, size, va, and the owner and system totals at
  the time) for diagnostics.

So a per-process quota question is answered from `OWNERS` and a system-pressure question from
`SYSTEM_BYTES`, separately. The honest limit is that the owner table holds `OWNER_CAP = 64`
processes and the event ring holds 20 entries (`state.rs:20`, `:21`), so both are bounded
snapshots, not an unbounded ledger.

## Security analysis

`MkMmap` and `MkMunmap` are the only calls that let a capsule add or remove pages in its own
space, so their properties are about staying inside that space and leaving no residue.

**A capsule can only touch its own user space.** Both calls bound `length` at `MAX_MMAP_SIZE`
and require any fixed address to lie within user space via `is_user_space` (`consts.rs:25`),
which also rejects an `addr + len` overflow, so neither path can be steered onto a kernel
address. The base of an allocator-owned mapping comes from the caller's per-process `MmapVa`
(`reserve_va.rs:19`), never a global pool, so one capsule cannot reserve into another's range.
The capability gate requires `Memory` for `MkMmap` and its deallocate side for `MkMunmap`
(`cap_table/mk.rs`), so a capsule without the memory bit cannot map at all.

**No page carries a previous tenant, and no partial range survives a failure.** Every mapped
page is zeroed with `zero_frame` before the call returns (`mmap.rs:87`), and every unmapped
frame is zeroed by `deallocate_frame` on the way back to the allocator, so mmap memory is clean
on both entry and exit. A frame-allocation or mapping failure mid-map rolls back every page
already installed and releases the reserved VA (`mmap.rs:70`, `:82`) before returning `ENOMEM`,
so a failed mmap is atomic.

**W^X holds even though the syscall does not check it.** The handler forwards `PROT_WRITE` and
`PROT_EXEC` into the permission set unchanged; the enforcement is the `map_page` gate, which
refuses a writable-executable PTE with `WXViolation`. So a capsule asking for `RWX` gets the
mapping refused at the page-table install, not a silently downgraded or silently granted page.
The honest boundary is that mmap installs no guard pages between mapped regions: the hardening
guard-page tracker exists but nothing registers with it (see [hardening](hardening.md)), so an
overrun off one mmap region into an adjacent mapped one is not tripwired here.

## Debugging mmap and munmap

The return value localises a failure directly. `EINVAL` is a bad argument: a zero or oversize
length, an unaligned fixed address, or, on the fixed path, an overlap with an already-mapped
page. `EPERM` on `MkMmap` is a fixed address outside user space (distinct from the capability
`EPERM` the gate raises before the handler runs). `ENOMEM` is exhaustion: either the
per-process VA allocator had no run of `pages` free (`reserve_va` returned `None`) or the frame
allocator ran out mid-map, in which case the rollback already ran and the space is unchanged.
The `[MMAP] release_va_failed` / `[MUNMAP] frame_release_failed` serial lines
(`mmap.rs:72`, `munmap.rs:44`) are internal-consistency alarms: they mean a rollback or free
step itself failed, which is a bug in the VA or frame bookkeeping rather than a caller error.
For footprint questions, read `SYSTEM_BYTES` for the whole-system total and the `OWNERS` entry
for a specific pid; a pid whose `OWNERS` bytes climb without matching `record_munmap` events in
the ring is leaking mappings.

## Source map

```
  src/syscall/microkernel/memory/mmap.rs        sys_mmap
  src/syscall/microkernel/memory/munmap.rs      sys_munmap
  src/syscall/microkernel/memory/consts.rs      MAX_MMAP_SIZE, is_user_space, PROT_* bits
  src/syscall/microkernel/memory/va/            reserve_va, release_va, rollback_mapped_pages
  src/syscall/microkernel/memory/accounting/    record_mmap, record_munmap, OWNERS/SYSTEM_BYTES/EVENTS
```

Every reference above is verified against those trees. The `map_page`/`unmap_page` gate these
calls funnel through and its W^X check are on the [paging manager](paging-manager.md) and
[hardening](hardening.md) pages, the frames come from and return to the
[physical frame allocator](physical-frames.md), the zero-on-free that scrubs an unmapped page
is on the [zeroization](zeroization.md) page, and the `MkMmap`/`MkMunmap` ABI and capability are
on the [syscall reference](../../abi/syscalls.md).

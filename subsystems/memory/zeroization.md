# RAM Residency and Zeroization

The defining property of NØNOS is that it runs from memory and leaves nothing
behind. There is no disk in the trust path, memory is scrubbed as it is reclaimed
during normal operation, and a single whole-system routine erases everything on a
ZeroState event. This page states that guarantee as the code actually implements
it: what is zeroed, when, and by which pass, and just as importantly where the
guarantee is single-pass zeroing versus a multi-pass secure erase, so the claim is
precise rather than a slogan.

## RAM-resident by architecture

The system boots, verifies, and runs entirely from memory. Capsules are not read
off a filesystem; they are signed artifacts compiled into the kernel image and
verified before they run, as the [verified-spawn gate](../../security/capsules-and-trust.md)
describes, so there is no on-disk image in the trust path to leave behind. What a
user chooses to keep is the exception, and it is explicit: it goes through the
encrypted store with recorded consent, and that consent is one of the things the
wipe below revokes. Everything else is transient by construction, and the
zeroization paths below make that concrete.

## Continuous zeroization

In normal operation, memory is scrubbed as it is freed, so a later allocation never
sees an earlier one's data. Four paths do this, each verified in its own module.

Every physical frame is zeroed when it is returned to the frame allocator. The
higher-level `deallocate_frame` calls `zero_frame` before the frame is reusable
(`src/memory/frame_alloc/manager/alloc.rs:41`), and `zero_frame`
(`src/memory/frame_alloc/manager/zero.rs:21`) bounds-checks the frame against the
direct map and writes 4 KiB of zeros through it. So a frame that held a capsule's
data is zero by the time any other allocation can receive it.

The [heap](heap.md) zeroes on both allocation and free: `HEAP_ZERO_ON_ALLOC` and
`HEAP_ZERO_ON_FREE` both default to on, so allocations start zeroed and freed blocks
are wiped as they are returned.

The [fault handler](faults.md) zeroes a demand-backed page before mapping it, so a
lazily populated page never exposes the previous contents of the frame it drew.

Secure memory regions are zeroed on deallocation, scaled by their security level.
`deallocate_region` (`src/memory/secure_memory/manager/dealloc.rs:29`) calls
`secure_zero_memory(va, size, security_level)` before it releases the virtual range,
so a region allocated at a higher security level gets a correspondingly stronger
erase when it is freed.

## How a capsule's memory is reclaimed

A capsule leaves no residue not because of a dedicated scrub in the exit path, but
because of the frame-free zeroing above. When a capsule exits, its address space is
torn down and its physical frames are returned to the allocator, and each of those
frames is zeroed on return by `zero_frame`. The guarantee that the next tenant of
that memory sees only zeros is therefore the composition of teardown returning the
frames and the allocator zeroing them, which is exactly the property the Lean
`Zeroization` module proves at the specification level: a wiped region reveals
nothing of what it held, and a reused region leaks nothing across lifetimes. The
page states this as it is: the mechanism is frame-free zeroing, not a separate exit
wipe, and that mechanism is sufficient because no freed frame is reusable until it
is zeroed.

## The ZeroState wipe

The explicit whole-system erase is now wired, and it fires on every clean exit.
`terminate` (`src/security/zerostate/terminate.rs:35`) is described in its own source
as "the only way out of a running system," and both the shutdown syscall
(`src/syscall/dispatch/router/admin/shutdown.rs:26`) and the reboot syscall
(`admin/reboot.rs:27`) reach the firmware only through it:

```
  terminate(off):                                          # terminate.rs:35
      broadcast_ipi(Ipi::Stop)      # 1. quiesce the other CPUs (best effort)
      zerostate_shutdown_wipe()     # 2. wipe process memory, stacks, keys, heap
      enter(off)                    # 3. hand to firmware; never returns
```

The order is the whole point and the source says so (`terminate.rs:23`). The other
CPUs are stopped first, because a core still scheduling would write memory behind the
wipe and that write would survive into the next boot; quiescing first makes the wipe a
snapshot rather than a race. The stop is best effort by design: on the shipping
single-core boot there is nobody to stop, and on a controller that refuses the
broadcast the honest choice is to wipe what this core can reach rather than stay up
holding everything (`terminate.rs:31`).

`zerostate_shutdown_wipe` (`src/security/hardening/memory_sanitization/api.rs:59`) does
the erase, and its own step ordering is load-bearing:

```
  zerostate_shutdown_wipe():                               # api.rs:59
      raise SANITIZATION_LEVEL to Paranoid
      for each process: sanitize_process_memory(pid)       # code region + every VMA
      wipe_kernel_stacks()                                 # page-allocator stacks
      fs::clear_caches()                                   # fs + cryptofs state
      crypto::vault::zeroize_all_keys()                    # while the heap is readable
      restore SANITIZATION_LEVEL
      dod_5220_erase(heap_start, heap_size)                # LAST; covers the free list
```

It raises the sanitization level to `Paranoid` (`api.rs:63`), then walks every process
and wipes its code region and every VMA through the owner's address space
(`sanitize_process_memory`, `api.rs:37`, resolving the target asid and wiping through
the directmap rather than dereferencing another AS's addresses directly). It then wipes
the kernel stacks, which come from the page allocator and are reached by neither the
heap erase nor the process wipe (`api.rs:73`); clears the filesystem and cryptofs
caches (`api.rs:78`); and zeroes the crypto key vault while the heap is still readable
(`api.rs:82`). The heap erase runs last (`api.rs:96`) because it covers the allocator's
own free list and nothing may allocate after it; `terminate` calls into the firmware
from there, which does not allocate. The heap extent comes from the allocator's own
`extent()`, not a static layout constant, which the comment records as a fix for a bug
where the wipe erased an unmapped range while every heap-resident secret stayed in DRAM
(`api.rs:91`).

## The multi-pass erase

The heap erase uses `dod_5220_erase`
(`src/security/hardening/memory_sanitization/erase.rs:62`), a multi-pass overwrite
modelled on the DoD 5220.22-M pattern. The same module provides the steady-state
`secure_zero` (`erase.rs:26`), the level-driven `sanitize` (`erase.rs:163`) that the
free path calls, and the stronger `paranoid_erase` (`erase.rs:105`) and `gutmann_erase`
(`erase.rs:127`) that `Paranoid` level selects. The DoD pattern writes multiple volatile
passes so the compiler cannot elide them, ending on a zero pass. This is the strong
erase; the continuous zeroing above is a single zero pass, which is what steady-state
reclaim needs, and the multi-pass pattern is reserved for the whole-system ZeroState
wipe at shutdown.

## What this does and does not claim

Stated precisely: freed memory is single-pass zeroed as it is reclaimed, so no
allocation sees a previous one's data, and this runs on every free; a capsule's
frames are zeroed as its address space is torn down on exit, so it leaves no
readable residue; secure regions get a stronger erase on free; and the whole-system
ZeroState wipe is a multi-pass erase of process memory, kernel stacks, caches, keys,
and the heap that now fires on every clean exit, because both the shutdown and reboot
syscalls reach the firmware only through `terminate`, which wipes before it powers off.
The remaining honest gap is coverage, not wiring: the wipe runs on an orderly shutdown
or reboot, but a hard power loss or a panic that does not route through `terminate`
gets only whatever continuous zeroing already happened, and the CPU-stop that precedes
the wipe is best effort. What the software wipe addresses is data remanence in memory
the kernel can address, by overwriting it and flushing it out of cache. It does not
claim to defeat physical attacks below that level, such as cold-boot DRAM remanence
against removed memory, which is a hardware property outside the wipe's reach. The
guarantee is that nothing the kernel can address is left readable, and that no freed
memory is reused before it is zeroed.

## Security analysis

The whole page is a security argument, so this states it as named properties and, crucially, where the
guarantee is a running mechanism versus an unwired capability.

**No cross-lifetime leakage in steady state.** Every physical frame is zeroed on return to the
allocator by `zero_frame` (`frame_alloc/manager/zero.rs:21`), which bounds-checks the frame against the
direct map before writing 4 KiB of zeros, so a frame that held a capsule's data is zero by the time any
other allocation can receive it. The [heap](heap.md) zeroes on both alloc and free, the
[fault handler](faults.md) zeroes a demand-backed page before mapping it, and secure regions get an
erase scaled by security level on free (`secure_memory/manager/dealloc.rs:29`). This is the property the
Lean `Zeroization` module proves at the spec level, and it is a *running* mechanism: it fires on every
free, so a capsule's frames are scrubbed as its address space is torn down on exit, with no dedicated
exit wipe needed.

**The whole-system erase is strong and wired to every clean exit.** `terminate`
(`src/security/zerostate/terminate.rs:35`) is the only exit from a running system, reached by both the
shutdown (`admin/shutdown.rs:26`) and reboot (`admin/reboot.rs:27`) syscalls. It stops the other CPUs,
runs `zerostate_shutdown_wipe` (`memory_sanitization/api.rs:59`), then hands off to firmware. The wipe
raises the sanitization level to `Paranoid`, wipes every process's code and VMAs through the owner's
address space, wipes the page-allocator kernel stacks, clears fs and cryptofs caches, zeroes the crypto
vault while the heap is still readable, and multi-pass erases the heap last with `dod_5220_erase`
(`erase.rs:62`) over the allocator's real `extent()` so it covers the free list. The honest boundary is
now coverage rather than wiring: the wipe fires on an orderly shutdown or reboot, but a hard power loss
or a panic that does not route through `terminate` falls back to whatever continuous zeroing had run, and
the CPU-stop preceding the wipe is best effort (nobody to stop on a single-core boot, a refusable
broadcast on a multi-core one).

**The claim is bounded to memory the kernel can address.** The software wipe defeats data remanence in
addressable memory by overwriting and flushing it. It does not claim to defeat physical attacks below
that level, such as cold-boot DRAM remanence against removed memory, which is a hardware property
outside its reach. The guarantee is precisely that nothing the kernel can address is left readable, and
that no freed memory is reused before it is zeroed.

## Debugging zeroization

Zeroization is mostly invisible when it works, so debugging is about knowing which pass ran and what its
one console signal means. The continuous zeroing is silent: `zero_frame` returns without writing if the
frame is outside the direct map (`zero.rs`), which is the one way a free could *not* scrub, so a stale
byte surviving a free points at a frame whose physical address fell outside `DIRECTMAP_SIZE` rather than
at a missing zero pass. The whole-system wipe narrates its bounds: it logs
`"[SANITIZE] ZeroState shutdown wipe initiated"` and, after the heap erase,
`"[SANITIZE] ZeroState shutdown wipe complete"` (`api.rs:60`, `:85`), so a shutdown or reboot that shows
the first line but not the second wedged inside the wipe (most likely in a per-process range or the vault
walk), while neither line on a clean exit would mean `terminate` was bypassed. When reasoning about a suspected leak, the
distinction that matters is which mechanism should have covered it: a reused *frame* is the allocator's
`zero_frame`, a reused *heap block* is `HEAP_ZERO_ON_FREE`, a *demand page* is the fault handler's zero,
and a *secure region* is `secure_zero_memory`. The whole-system DoD erase is only for a ZeroState event
and is not part of any of those steady-state paths.

## Verification

The specification-level `Zeroization` proofs in the
[verification stack](../../architecture/verification.md) prove that a wiped region
holds no secret and that a reused region leaks nothing across lifetimes, which is the
abstract statement of the frame-free zeroing and the ZeroState wipe documented here.

## Source map

```
  src/memory/frame_alloc/manager/alloc.rs      deallocate_frame calls zero_frame
  src/memory/frame_alloc/manager/zero.rs       zero_frame, the direct-map zero pass
  src/memory/heap/manager/globals.rs           HEAP_ZERO_ON_ALLOC / HEAP_ZERO_ON_FREE
  src/memory/secure_memory/manager/dealloc.rs  secure_zero_memory on region free
  src/security/zerostate/terminate.rs           terminate: stop CPUs, wipe, power off
  src/syscall/dispatch/router/admin/{shutdown,reboot}.rs  the two callers of terminate
  src/security/hardening/memory_sanitization/api.rs    zerostate_shutdown_wipe, sanitize_process_memory
  src/security/hardening/memory_sanitization/erase.rs  dod_5220_erase, sanitize, secure_zero
```

Every reference above is verified against those trees. The demand-page zero is on the
[fault handler](faults.md) page, the heap zeroing is on the [heap](heap.md) page, the frame-free zero is
the free-path arm of the [physical frame allocator](physical-frames.md), and the spec proofs are on the
[verification stack](../../architecture/verification.md) page.

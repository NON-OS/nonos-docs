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

The explicit whole-system erase is `secure_wipe_all_memory`
(`src/security/wipe.rs:23`). It is a complete and correct erase routine, and it does
considerably more than zero the heap:

```
  secure_wipe_all_memory():
      revoke_all_consent()          drop every persistence consent
      wipe_heap_region()            DoD 5220 multi-pass erase of the heap
      wipe_process_memory()         zero every process's code region and VMAs
      wipe_crypto_keys()            delete every key in the crypto vault
      wipe_ipc_buffers()            reinitialise IPC, clearing its buffers
      wipe_vfs_caches()             clear the filesystem caches
      compiler_fence(SeqCst)
      log "ZeroState secure memory wipe complete"
```

It first revokes all persistence consent so nothing can be written out during or
after the wipe, then erases the heap with the multi-pass routine below, then walks
every process and volatile-zeros its code region and every virtual memory area,
then deletes all crypto vault keys, reinitialises IPC to clear its buffers, and
clears the VFS caches. The per-process and per-region sizes are bounds-checked
before the wipe so a corrupt PCB cannot direct the wipe outside a sane range.

One thing has to be stated plainly, because the code is unambiguous about it: as
the tree stands, `secure_wipe_all_memory` has no caller. A search of the whole
repository finds only its definition; nothing in the kernel invokes it, not the
shutdown path, not the panic handler, not a tamper response. The routine is
therefore a capability, not an automatic guarantee. What enforces ephemerality in
operation is the continuous zeroing above, which does run, on every free. The
one-shot whole-system erase is built and correct but not wired, and making it fire
on its own would mean calling it from the shutdown, panic, and tamper paths. This
page records that gap rather than describing a wipe that does not currently
trigger.

## The multi-pass erase

The heap wipe uses `dod_5220_wipe` (`security/wipe.rs:44`), a multi-pass overwrite
modelled on the DoD 5220.22-M pattern, with verification and a cache flush:

```
  dod_5220_wipe(data):
      pass 1: write 0x00 to every byte (volatile)
      pass 2: write 0xFF to every byte (volatile)
      pass 3: write secure-random bytes
      pass 4: write 0x00 to every byte (volatile)
      verify_wipe(data)                     read back, warn and re-wipe if nonzero
      flush_cache_lines(data)               clflush every line, then mfence  (x86_64)
```

Each pass is a volatile write so the compiler cannot elide it, separated by
sequential-consistency fences. The random pass draws from the secure RNG. After the
final zero pass it reads the region back to verify the wipe took, and on x86_64 it
flushes every cache line with `clflush` and an `mfence`, so the zeros reach memory
rather than sitting in cache. This is the strong erase; the continuous zeroing above
is a single zero pass, which is what steady-state reclaim needs, and the DoD pattern
is reserved for the whole-system ZeroState wipe.

## What this does and does not claim

Stated precisely: freed memory is single-pass zeroed as it is reclaimed, so no
allocation sees a previous one's data, and this runs on every free; a capsule's
frames are zeroed as its address space is torn down on exit, so it leaves no
readable residue; secure regions get a stronger erase on free; and the whole-system
`secure_wipe_all_memory` is a multi-pass, verified, cache-flushed erase, but it is
implemented and not wired to any trigger, so it is a capability that does not fire on
its own today. The operational ephemerality guarantee rests on the continuous
zeroing, which does run; the one-shot whole-system wipe is a gap to close by
invoking it from the shutdown, panic, and tamper paths. What the software wipe addresses is data remanence in memory
the kernel can address, by overwriting it and flushing it out of cache. It does not
claim to defeat physical attacks below that level, such as cold-boot DRAM remanence
against removed memory, which is a hardware property outside the wipe's reach. The
guarantee is that nothing the kernel can address is left readable, and that no freed
memory is reused before it is zeroed.

## Verification

The specification-level `Zeroization` proofs in the
[verification stack](../../../verification/README.md) prove that a wiped region
holds no secret and that a reused region leaks nothing across lifetimes, which is the
abstract statement of the frame-free zeroing and the ZeroState wipe documented here.

## Source

```
  src/memory/frame_alloc/manager/alloc.rs   deallocate_frame calls zero_frame
  src/memory/frame_alloc/manager/zero.rs      zero_frame, the direct-map zero pass
  src/memory/heap/manager/globals.rs           HEAP_ZERO_ON_ALLOC / HEAP_ZERO_ON_FREE
  src/memory/secure_memory/manager/dealloc.rs  secure_zero_memory on region free
  src/security/wipe.rs                          secure_wipe_all_memory and dod_5220_wipe
```

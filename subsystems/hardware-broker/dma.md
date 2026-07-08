# DMA Grants

A device that does bus-master DMA needs memory it can reach by physical address: descriptor
rings, queues, framebuffers. `MkDmaMap` allocates that memory, zeroes it, maps it into the
capsule for CPU access, and hands back both the user virtual address and the device-visible
physical address. The size a capsule can request is capped per device class. This page
documents the path and its limits. The code is under `src/hardware/broker/dma/`.

## The mapping path

`map_for_caller` (`src/hardware/broker/dma/map/mod.rs:27`) is a four-step transaction, and it
owns the rollback chain so a failure at any step leaves nothing behind:

```
  map_for_caller(pid, req):
      claim_epoch = validate(req, pid)         else fail
      pages       = req.length / PAGE_SIZE
      phys_start  = alloc_and_zero(pages)       else fail
      user_va     = install(pages, phys_start)  else { free(phys_start); fail }
      record DmaGrant { grant_id, pid, device_id, claim_epoch, phys_start, user_va, length }
      return { user_va, device_addr: phys_start, length, grant_id }
```

`validate` runs the same claim and epoch check as the other grant classes and additionally
bounds the length against the class ceiling. The frames are allocated and zeroed before the
capsule ever sees them, so a DMA buffer never hands the device or the capsule a previous
tenant's bytes. The `install` step maps the frames into the capsule's address space for CPU
access; if it fails, the just-allocated frames are freed before returning, which is why the
allocation and the install are split into their own files with the top function as the
transaction boundary. The returned `device_addr` is the physical start the device programs into
its descriptors.

## The per-class ceiling

A capsule cannot request an unbounded DMA region. `dma_page_limit_for_class`
(`dma/limits.rs:31`) sets a page ceiling per device class, and a request over its class ceiling
is rejected with `BadLengthForClass`:

```
  RNG / INPUT / SERIAL   1 page          NETWORK              64 pages
  AUDIO                  16 pages         USB host (xHCI)      256 pages
  BLOCK                  1024 pages       DISPLAY              8192 pages
  anything unclassified  16 pages (the conservative fallback)
```

The ceilings are sized to the real need of each class, a network descriptor ring or an NVMe
submission and completion queue fits inside its ceiling, a random-number descriptor does not
need more than a page, and a display primary surface is framebuffer sized (8192 pages covers one
3840x2160 ARGB surface). The point is that a misbehaving capsule pays the cost of its own
over-request at its own class ceiling rather than being able to exhaust physical memory through
the DMA path.

## Records and revocation

The grant is recorded as a `DmaGrant` (`dma/types.rs:20`) carrying the grant id, holder,
device, epoch, physical start, user VA, and length. Revocation mirrors the MMIO paths:
`unmap_grant`, `release_for_device`, and `release_all_for_pid` (`dma/release.rs`), each of which
unmaps the user pages and frees the frames, with the self-context decision described on the
[revocation](revocation.md) page. Because the frames are broker-allocated (not device BAR
memory), releasing a DMA grant returns real RAM to the frame allocator.

## Source

```
  src/hardware/broker/dma/map/mod.rs       the transaction boundary and rollback
  src/hardware/broker/dma/map/validate.rs  claim, epoch, alignment, class-length checks
  src/hardware/broker/dma/map/alloc.rs     allocate and zero frames
  src/hardware/broker/dma/map/install.rs   map the frames into the capsule
  src/hardware/broker/dma/limits.rs        the per-class page ceilings
  src/hardware/broker/dma/release.rs       revocation
```

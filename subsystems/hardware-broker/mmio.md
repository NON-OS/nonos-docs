# MMIO Grants

A driver capsule needs to reach its device's registers, which live behind a physical BAR. The
broker is the one in-kernel path that maps a slice of a device BAR into a capsule's address
space, and it does so only for the claim holder, only within the BAR, and never over the
device's MSI-X table. This page documents `MkMmioMap` and its revocation. The code is under
`src/hardware/broker/mmio/` and `src/hardware/broker/grant.rs`.

## The mapping path

`map_for_caller` (`src/hardware/broker/mmio/map.rs:44`) is the whole path, five ordered steps,
and on any rejection no mapping is installed and no record is made:

```
  1. reject unknown flags, zero length, unaligned offset/length
  2. resolve the caller's claim; verify pid and epoch (StaleEpoch)
  3. resolve the device and BAR; verify it is an MMIO BAR
  4. compute [phys_start, phys_end); verify it is contained in the BAR
  5. clamp against the MSI-X table; reserve user VA; install pages; record
```

Every arithmetic step is checked: `phys_start = bar.base + offset`, `phys_end = phys_start +
length`, and the BAR end are all `checked_add`, and `phys_end > bar_end` is `BadRange`. A
request can only ever map memory that is inside the BAR of a device the caller holds; it cannot
name an arbitrary physical address, because the physical base comes from the kernel's device
table, not from the request. The offset and length must be page aligned, and the BAR base
itself must be page aligned, so a mapping never straddles a page it should not.

## The user window

The mapped pages land in a dedicated per-capsule MMIO virtual region, `[USER_MMIO_BASE,
USER_MMIO_END)` (`grant.rs:135`), carved by `reserve_user_va` (`grant.rs:145`). The allocator
adds a guard page between adjacent grants (`grant.rs:147`), so a runaway access off the end of
one grant faults into an unmapped page rather than spilling into the next grant. The pages are
installed by `map_user_mmio` with user, read-write, uncached, and no-execute attributes
(`map.rs:100`): a device register window is data the capsule reads and writes, never code it
executes, and uncached because it is device memory, not RAM.

## The MSI-X exclusion

A device's MSI-X interrupt table often shares a BAR with its registers. Exposing that table to
the capsule would let it program its own interrupt vectors and bypass the [IRQ](irq.md) bind
allowlist, so the mapping is clamped to stop at the page below the table. `safe_length`
(`mmio/msix_exclusion.rs:35`) computes the clamp: a request that overlaps the MSI-X table or its
pending-bit array is trimmed to end at the page boundary before the protected region, and a
request that starts inside the protected region clamps to zero length and is refused with
`WouldExposeMsixTable`. A device like xHCI whose registers share the BAR still maps everything
up to the table; only the table pages themselves are withheld, because the kernel programs them
on the capsule's behalf during MSI-X bind.

## The grant record and revocation

The successful mapping is recorded as an `MmioGrant` (`grant.rs:37`) in the global grant table,
carrying the grant id, the holder pid, the device, the claim epoch, the physical start, the user
VA, and the length. That record is the authority for undoing the mapping later. Three revocation
entry points exist (`mmio/release.rs`): `unmap_grant` for a single grant by the holder,
`release_for_device` for every grant on one device, and `release_all_for_pid` for every grant a
pid owns. Each unmaps the user pages when the holder's address space is the active one and skips
the unmap otherwise, letting address-space teardown drop the page-table entries wholesale; the
[revocation](revocation.md) page covers that self-context decision and the exit wiring.

## Source

```
  src/hardware/broker/mmio/map.rs             the five-step mapping path
  src/hardware/broker/mmio/msix_exclusion.rs  the MSI-X table clamp
  src/hardware/broker/mmio/release.rs         unmap_grant / release_for_device / release_all_for_pid
  src/hardware/broker/grant.rs                the MmioGrant table and the user VA region
```

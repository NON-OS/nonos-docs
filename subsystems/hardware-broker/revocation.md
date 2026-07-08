# Revocation and Lifecycle

Every grant the broker issues is revocable, and a capsule that exits with grants still held does
not leak them. This is the property that keeps device authority from outliving the capsule that
holds it: a claim, and every MMIO, DMA, IRQ, and PIO grant that depended on it, is torn down when
the capsule releases the device, and unconditionally when the capsule dies. This page ties the
per-class revocation paths together. The code is in each class's `release` module and in
`src/process/exit/`.

## Three ways a grant ends

Each grant class exposes the same three revocation entry points:

```
  unmap_grant / release_grant   one grant, by the holder pid (an explicit capsule request)
  release_for_device            every grant tied to one device (MkDeviceRelease)
  release_all_for_pid           every grant a pid still holds (capsule exit)
```

All three enforce holder ownership: a single-grant revoke checks the requesting pid is the
grant's holder (`NotHolder` otherwise), and the device and pid drains match on the holder. A
capsule can only revoke its own grants. The `drain_*` helpers remove the matching records and
return them so the caller can undo the underlying resource, unmap the MMIO or DMA pages, free the
DMA frames, unbind the IRQ vector, forget the PIO window.

## The self-context decision

Unmapping a user page requires touching the holder's page tables, which are only directly
reachable when the holder's address space is the active one. The revocation paths handle this with
a self-context flag (`mmio/release.rs:45`):

```
  release_all_for_pid(pid, unmap_pages):
      drained = drain_for_pid(pid)
      if unmap_pages:  unmap each grant's user pages   // holder is the active address space
      drop the pid's device claims
```

When the holder is current, the pages are unmapped directly and the TLB is shot down. When the
holder is a different address space (a cross-pid teardown), the unmap is skipped and the
address-space teardown drops the page-table entries wholesale, which is both correct and cheaper
than walking a foreign address space page by page. The MMIO release also drops the pid's device
claims in the same call, so the claim and its dependent grants disappear together.

## Exit wiring

The exit path revokes all four grant classes. Process teardown calls each class's
`release_all_for_pid` (`src/process/exit/teardown.rs:33`):

```
  broker::release_all_for_pid(pid, current)    // MMIO grants + device claims
  broker::irq_release_all_for_pid(pid)         // IRQ bindings
  broker::dma_release_all_for_pid(pid, current) // DMA grants + frames
  broker::pio_release_all_for_pid(pid)          // PIO windows
```

The `current` flag is the self-context decision above: teardown passes whether the dying capsule's
address space is active so the MMIO and DMA paths know whether to unmap directly. The finalize
path (`src/process/exit/finalize.rs:18`) runs the same four releases with `unmap_pages = false`,
covering the case where the address space is already gone. Either way, a capsule cannot exit while
still holding a device: the claim is dropped, the register windows are unmapped, the DMA frames
return to the allocator, and the interrupt vectors are unbound. Combined with the epoch check on
[claim](claim.md), this means device authority is bounded by the life of the holder and by the
current ownership epoch, never longer.

## Source

```
  src/hardware/broker/mmio/release.rs   MMIO revocation and the self-context unmap
  src/hardware/broker/dma/release.rs    DMA revocation
  src/hardware/broker/irq/release.rs    IRQ unbind and MSI-X teardown
  src/hardware/broker/pio/*             PIO drain helpers
  src/process/exit/teardown.rs          the four-class revoke on exit
  src/process/exit/finalize.rs          the finalize-path revoke
```

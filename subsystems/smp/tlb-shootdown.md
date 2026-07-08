# SMP: TLB Shootdown

When a mapping is removed on one CPU, every other CPU that shares the address space may
still hold that translation cached in its TLB. Until those stale entries are
invalidated, another CPU could keep reading or writing through a page that has been
unmapped. A TLB shootdown is the cross-CPU invalidation that closes that window: the CPU
that changed the mapping tells the others to drop the entry and waits for them to
confirm. This page documents the shootdown broadcast, the IPI handler that services it,
and the local invalidation primitives. The code is `src/smp/tlb.rs`.

## The broadcast

`tlb_shootdown` (`src/smp/tlb.rs:22`) invalidates one virtual address across every CPU:

```
  tlb_shootdown(addr):
      if only one CPU is online:
          invalidate the page locally and return         no IPI needed
      publish addr, reset the ack count, mark a shootdown active
      send the TLB-shootdown IPI to all other CPUs
      invalidate the page locally
      wait until (cpus_online - 1) CPUs have acked, or a timeout elapses
      mark the shootdown inactive
```

On a single-CPU system it degrades to a plain local `invlpg` with no inter-processor
traffic. With more than one CPU online it publishes the target address, resets the
acknowledgement counter, marks a shootdown active, and sends the shootdown
inter-processor interrupt to every other CPU. It invalidates the address in its own TLB,
then spin-waits until the number of acknowledgements equals the number of other online
CPUs. The wait has a bound: after roughly ten million TSC cycles it logs
`TLB shootdown timeout` and proceeds rather than hanging the CPU forever if one core is
unresponsive. When all others have acked, the shootdown is marked inactive.

## The IPI handler

Each other CPU services the shootdown in its interrupt handler
(`tlb.rs:57`):

```
  handle_tlb_shootdown_ipi():
      if a shootdown is active:
          read the published address
          invalidate that page locally
          increment the ack count
```

The receiving CPU reads the published address, invalidates it in its own TLB, and bumps
the acknowledgement counter that the initiator is waiting on. Because the initiator only
proceeds once every other CPU has acked, the mapping is guaranteed dropped from every
TLB before the initiator continues, so no CPU can use the stale translation afterward.

## The invalidation primitives

Two primitives do the actual work (`tlb.rs:68`). `invalidate_page` executes `invlpg`,
which removes a single page's translation from the TLB, and it is what both the initiator
and the IPI handler call for a targeted flush. `flush_tlb` reloads `CR3` from itself,
which flushes the entire non-global TLB at once, used where a whole-address-space flush
is cheaper than invalidating pages one at a time. The single-page path is the common
case; the full flush is for bulk changes.

## How the address-space scope is decided

`tlb_shootdown` itself sends to every other CPU. The decision of whether a given
unmapping needs a shootdown at all, and against which CPUs, is made a layer up, in the
[paging manager](../memory/paging-manager.md)'s per-ASID shootdown wrappers, which read
each CPU's [`active_asid`](per-cpu.md) to filter targets: a user-address flush skips a CPU
that is not running the affected address space, while a kernel-address flush reaches
every online CPU because kernel mappings are shared by all of them. This file is the
broadcast-and-wait primitive; the paging manager decides when to invoke it.

## Source

```
  src/smp/tlb.rs        tlb_shootdown, handle_tlb_shootdown_ipi, invalidate_page, flush_tlb
  src/smp/ipi/          the inter-processor interrupt send and vectors
  src/smp/state.rs      the shootdown address, ack, and active flags
```

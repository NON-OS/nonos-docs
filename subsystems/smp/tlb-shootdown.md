# SMP: TLB Shootdown

When a mapping is removed on one CPU, every other CPU that shares the address space may
still hold that translation cached in its TLB. Until those stale entries are
invalidated, another CPU could keep reading or writing through a page that has been
unmapped. A TLB shootdown is the cross-CPU invalidation that closes that window: the CPU
that changed the mapping tells the others to drop the entry and waits for them to
confirm. This page documents the shootdown broadcast, the IPI handler that services it,
the local invalidation primitives, and the fail-hard timeout policy. The code is
`src/memory/paging/manager/shootdown.rs`; there is no `src/smp/tlb.rs`.

Every path here is dormant on the shipping x86_64 build, which is single-core
(`cpus_online() == 1`), so the broadcast block is never entered and only the local
`invlpg`/`invalidate_all` runs. The code is complete and this page describes it, but the
cross-CPU handshake is not exercised on the default build.

## The entry points

Three public functions front the shootdown, one per mutation shape
(`shootdown.rs:52`, `:61`, `:80`):

```
  flush_tlb_one_smp(va, asid)                single page
  flush_tlb_range_smp(start, page_count, asid)   a run of pages
  flush_tlb_all_smp(asid)                    the whole non-global TLB
```

Each does the local invalidation first and unconditionally, then returns early if
`cpus_online() <= 1` (`shootdown.rs:54`, `:73`, `:82`); only on a multi-CPU runtime does
it fall through to `broadcast`. `flush_tlb_range_smp` promotes a run of more than 32 pages
to a full flush (`shootdown.rs:65`) rather than issue dozens of single-page IPIs, and it
encodes a whole-TLB flush as `page_count == 0` in the request slot, which the receiver
reads as `invalidate_all` (`shootdown.rs:85`, `:165`).

## The broadcast and the ack countdown

`broadcast` (`shootdown.rs:90`) is the cross-CPU primitive. The handshake is a countdown,
not a count-up, and each target is gated by its own per-CPU flag rather than a global
"active" flag:

```
  broadcast(va, page_count, asid):
      acquire SHOOTDOWN_LOCK, servicing in-flight rounds while spinning (see below)
      for each other online CPU whose active_asid matches (cpu_should_flush):
          set that CPU's tlb_flush_pending = 1        # mark before publishing
          targets += 1
      if targets == 0: return
      publish REQ_VA, REQ_PAGES
      REQ_PENDING_ACKS = targets
      for each marked CPU: send_ipi(apic_id, Ipi::TlbShootdown)
      wait_for_acks()
```

Each target CPU is marked with `tlb_flush_pending = 1` before the request is published
(`shootdown.rs:118`), so a CPU already spinning in the handler cannot read the round as
"not mine." The initiator then publishes the address and page count with `Release`
stores, sets `REQ_PENDING_ACKS` to the target count with `SeqCst` (`shootdown.rs:125`),
and sends the `TlbShootdown` IPI to each marked CPU (`shootdown.rs:139`).

The asid filter is inside `broadcast`, not a layer up. `cpu_should_flush`
(`shootdown.rs:145`) reaches every online CPU for a kernel-half flush
(`asid == ASID_KERNEL == 0`) because the kernel half is shared by every address space,
and for a user asid reaches only CPUs whose `active_asid` currently equals it
(`shootdown.rs:149`). The paging-manager callers just pass the asid of the mapping they
changed; the target selection happens here.

## The re-entrancy guard

`broadcast` cannot simply block on `SHOOTDOWN_LOCK`. Page-table mutation sites reach it
with interrupts masked, so a CPU that blocked on the lock could not answer the current
holder's shootdown IPI, and the two would wait on each other until the timeout below
halted the machine. Instead the lock is taken in a loop that services any round already in
flight while it spins (`shootdown.rs:95`):

```
  loop:
      if SHOOTDOWN_LOCK.try_lock() succeeds: break
      handle_shootdown_ipi()      # answer the holder so it can finish
      spin_loop()
```

This is why `handle_shootdown_ipi` is safe to call both from the interrupt vector and
directly from this spin: the per-CPU `tlb_flush_pending` flag is what says a round applies
to this CPU, and clearing it before the ack means neither path can acknowledge twice.

## The IPI handler

Each target CPU services the shootdown through `handle_shootdown_ipi`
(`shootdown.rs:159`), reached from the `TlbShootdown` vector via the dispatch handler
`tlb_shootdown` (`src/smp/ipi_dispatch/handlers.rs:30`):

```
  handle_shootdown_ipi():
      if tlb_flush_pending.swap(0, AcqRel) == 0: return    # not my round, or already done
      if REQ_PAGES == 0: invalidate_all()
      else: invlpg each of REQ_PAGES pages from REQ_VA
      REQ_PENDING_ACKS.fetch_sub(1, Release)
```

The `swap(0)` is both the gate and the de-dup: a CPU whose flag is already zero (a
spurious IPI, or a round it already answered from the spin path) returns without touching
the counter. A CPU that was a genuine target invalidates the published range, then
decrements `REQ_PENDING_ACKS`. The initiator is waiting for that counter to reach zero.

## The invalidation primitives

The actual invalidation is `tlb::invalidate_page`
(`src/memory/paging/tlb/invalidate.rs:30`), which executes `invlpg` for one page, and
`tlb::invalidate_all` (`invalidate.rs:36`), which flushes the whole non-global TLB. Both
the initiator's local step and the IPI handler call these. The single-page path is the
common case; the full flush is for a range over 32 pages or an explicit whole-TLB request.

## The timeout policy: fail-hard

The wait is bounded, and the bound is fail-hard, not best-effort. `wait_for_acks`
(`shootdown.rs:177`) spins until `REQ_PENDING_ACKS` reaches zero, and if the deadline
(`SHOOTDOWN_TIMEOUT_TSC`, ~10M TSC cycles, `shootdown.rs:44`) passes with an ack still
outstanding it does not proceed:

```
  wait_for_acks():
      deadline = tsc() + SHOOTDOWN_TIMEOUT_TSC
      while REQ_PENDING_ACKS > 0:
          if tsc() > deadline:
              println("[FATAL] TLB shootdown timeout")
              send_panic_ipi()
              halt_loop()          # the machine stops here
          spin_loop()
```

The module header states the reasoning (`shootdown.rs:21`): a stale TLB entry would back
freed DMA or MMIO, so continuing while an ack is missing would let a CPU read or write
through a page that was unmapped. Rather than accept that, the initiator broadcasts a
panic IPI (`shootdown.rs:182`) and halts (`shootdown.rs:183`). Proceeding is the one thing
this path will not do. The tradeoff, stated in the same comment, is that the timeout must
be tuned upward if ever too tight, never tuned to "wait forever."

## Security analysis

A missed TLB shootdown is a memory-safety hole: a CPU that keeps a stale translation can read or write
through a page that was unmapped and possibly handed to another tenant. The properties here are about
never letting the initiator proceed while a stale entry survives. Three hold.

**The initiator waits for every target to acknowledge, or halts.** `broadcast` (`shootdown.rs:90`) counts
its targets, publishes the request, and `wait_for_acks` (`shootdown.rs:177`) spins until
`REQ_PENDING_ACKS` reaches zero. Each target decrements that counter only after it has invalidated the
published range (`shootdown.rs:174`). The publish uses `Release` stores on the address and page count and
the receiver reads them `Acquire`, so a target sees the address before it acts and the initiator sees the
decrements after they land. The initiator does not return until the count is zero, so on the success path
the mapping is dropped from every target TLB before it continues.

**The per-CPU pending flag makes a spurious IPI a no-op and de-dups the spin path.** `handle_shootdown_ipi`
(`shootdown.rs:159`) swaps its own `tlb_flush_pending` to zero and returns if it was already zero, so a
stray or late vector, or a round this CPU already answered while spinning for the lock, does nothing and
cannot double-decrement the ack counter. There is no global "active" flag to desynchronise; the round's
membership is per-CPU state set before the request is published.

**The timeout is fail-hard, so a wedged CPU cannot produce a silent stale translation.** This is the
correction to the earlier design, which described proceeding on timeout as best-effort safety. The code
does the opposite: on a missed ack it prints `[FATAL] TLB shootdown timeout` (`shootdown.rs:181`), sends a
panic IPI, and halts the originator (`shootdown.rs:182`). The honest boundary is availability, not safety:
a stuck core takes the machine down rather than letting the initiator run on with a possibly-stale peer
TLB. That is the deliberate choice, because the alternative is a use-after-free-shaped corruption on the
core that never flushed.

## Debugging TLB shootdown

The one message this path prints is fatal, and it is the anchor:

```
  [FATAL] TLB shootdown timeout    an ack never arrived within ~10M TSC cycles; the machine halted
```

**A shootdown timeout.** The initiator waited for its targets and one never decremented the counter,
then panicked and halted. The usual cause is a CPU not servicing the `TlbShootdown` vector: it took an
exception with interrupts off, it is spinning in a `cli` section that is not the re-entrancy loop, or its
IDT entry for the vector is not installed. Because `init_bsp` registers the IPI handlers
(`src/smp/init/bsp.rs:40` -> `register_ipi_handlers`, `src/smp/ipi_dispatch/x86_64.rs:55`) before any AP
starts, a timeout on a cleanly booted system points at a stuck core rather than a missing handler. Unlike
the old best-effort design, there is no "proceeded anyway" aftermath to chase: the machine is halted at
the timeout, and the panic IPI is the record.

**A stale translation with no timeout.** If a CPU reads through an unmapped page but nothing halted, the
shootdown either was scoped to skip that CPU wrongly, or was never a multi-CPU run at all. The asid filter
is `cpu_should_flush` (`shootdown.rs:145`): a CPU whose `active_asid` (`per-cpu.md`) is wrong, because a
context switch did not update it, would be excluded when it should have been a target. So a
stale-translation bug with no fatal points at the asid filter and the `active_asid` field, whereas a fatal
points at delivery and servicing. On the shipping single-core boot neither can occur: every entry point
degrades to a local `invlpg`/`invalidate_all` with no IPI when `cpus_online() <= 1`, so any shootdown bug
is a multi-CPU-only symptom that the current build does not reach.

## Source map

```
  src/memory/paging/manager/shootdown.rs   flush_tlb_{one,range,all}_smp, broadcast,
                                           cpu_should_flush, handle_shootdown_ipi, wait_for_acks
  src/memory/paging/tlb/invalidate.rs      invalidate_page (invlpg), invalidate_all
  src/smp/ipi_dispatch/handlers.rs         tlb_shootdown, the vector's dispatch handler
  src/smp/ipi_dispatch/x86_64.rs           register_ipi_handlers, the vector binding
  src/smp/init/bsp.rs                       init_bsp binds the IPI handlers before any AP starts
  src/smp/percpu/types.rs                   tlb_flush_pending and active_asid, the per-CPU state
```

Every reference above is verified against those trees. The `active_asid` field the filter reads to scope
targets is on the [per-CPU](per-cpu.md) page, the online-CPU count the wait depends on is established
during AP bring-up in the SMP init path, and the paging manager that calls these entry points is the
[paging manager](../memory/paging-manager.md).

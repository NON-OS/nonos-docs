# Interrupts and IO-APIC Routing

This page is the x86_64 interrupt routing that sits underneath the broker's IRQ
grants. When a driver capsule binds an interrupt, the kernel programs an IO-APIC
redirection entry, claims the line for that capsule, and allocates a delivery
vector. The [hardware broker page](hardware-broker.md) covers the driver-facing
syscalls; this page covers the routing table, GSI ownership, and the vector pool.

On aarch64 and riscv64 the same role is filled by those platforms' interrupt
controllers (a GIC, a PLIC) discovered through the device tree rather than ACPI.
The broker model above them is the same; only this routing layer differs.

---

## The routing table

The IO-APIC routing layer lives at
`src/arch/x86_64/interrupt/ioapic`. It is initialised during microkernel init,
after unified paging, by `init_from_acpi`, which reads the MADT to discover the
IO-APICs present and builds the `IOAPICS` table. It runs after paging because
programming a redirection entry is an MMIO write and the IO-APIC window is only
mappable once unified VM is up ([boot](boot.md), [memory](memory.md)).

A redirection table entry (RTE) is the per-line configuration the IO-APIC uses to
deliver an interrupt: the vector, the destination CPU, the delivery mode, and the
trigger and polarity flags. The kernel writes it as two 32-bit MMIO words
(`types_rte.rs`, `mmio.rs`).

---

## Programming a route

When the broker binds an INTx interrupt it calls `program_route_external`
(`src/arch/x86_64/interrupt/ioapic/ops_route.rs:94`):

```
  program_route_external(gsi, vector, dest_apic_id)
    claim_for_capsule(gsi)                 take ownership of the line
    rte = Rte::fixed(vector, dest_apic_id) build the redirection entry
    copy trigger and polarity flags from the MADT override, if any
    program_route(gsi, rte)                write the RTE over MMIO
    on MMIO failure, roll the ownership back
```

The trigger and polarity come from the MADT interrupt source overrides, so a line
that the firmware describes as level-triggered active-low is programmed that way
rather than assumed. If the MMIO write fails, the ownership claim is rolled back
so a failed bind does not leave the line owned by a capsule that is not driving
it.

---

## GSI ownership

Every global system interrupt has an owner, tracked as a per-line atomic
(`src/arch/x86_64/interrupt/ioapic/gsi_owners`):

```
  OWNER_FREE     0
  OWNER_KERNEL   1
  OWNER_CAPSULE  2
```

Ownership transitions are compare-and-swap, so two capsules racing to bind the
same line cannot both win (`gsi_owners/claim.rs`):

```
  claim_for_capsule(gsi)   CAS Free -> Capsule    (AcqRel)
  release_capsule(gsi)     CAS Capsule -> Free
```

A line the kernel owns cannot be claimed by a capsule, and a line one capsule
owns cannot be stolen by another. The CAS is what makes the broker's "claim the
device, then bind its IRQ" sequence safe under concurrency: the bind either takes
the free line or fails cleanly.

---

## The vector pool

Delivery vectors are allocated from a contiguous broker pool, leaving the rest of
the vector space for the kernel's own use
(`src/arch/x86_64/interrupt/broker/vectors.rs`):

```
  0x81..0xC0   broker IRQ pool        (BROKER_VEC_MIN .. BROKER_VEC_MAX)
  0xC1..0xF9   unused
  0xFA..0xFE   APIC LVT range
  0xFF         APIC spurious vector
```

The pool is contiguous so an MSI-X request for N vectors gets N consecutive
ones. A helper maps a vector back to its slot (`slot_of`) so the delivery path
can find the grant a firing vector belongs to. When an interrupt arrives on a
pool vector, the kernel bumps the owning grant's sequence counter and masks the
line, which is what the driver observes through `MkIrqPoll`, and `MkIrqAck`
unmasks it ([hardware broker](hardware-broker.md)).

---

## End to end

Putting the routing together with the broker:

```
  device raises GSI g
        |
        v
  IO-APIC delivers vector v (programmed at bind time) to the CPU
        |
        v
  kernel handler for v:  grant.seq += 1 ;  mask GSI g
        |
        v
  driver: MkIrqPoll sees seq advance, services the device
        |
        v
  driver: MkIrqAck -> unmask GSI g, ready for the next interrupt
```

The line is owned by exactly one capsule, programmed from the firmware's own
description of its trigger and polarity, delivered on a vector drawn from a pool
that cannot collide with kernel vectors, and held quiet between fire and
acknowledgement. That is the full chain a driver's interrupt travels, and every
step is gated by the device claim and the `Irq` capability above it.

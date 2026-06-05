# Hardware Broker

Drivers in NØNOS are capsules, and a capsule cannot touch hardware directly. The
broker is the narrow kernel surface that hands out grants for the four things a
driver needs: interrupts, memory-mapped IO, DMA, and on x86_64, port IO. Every
grant is gated by first claiming the device, which establishes an ownership
epoch. The [architecture overview](../architecture/overview.md) covers this in
section 12; the [interrupts page](interrupts.md) covers the IO-APIC side of IRQ
binding in detail.

---

## Claim first

Before any grant, a driver claims its device with `MkDeviceClaim`, which records
the owning pid and an epoch. The epoch is handed back and must be presented on
later grant calls, so a grant cannot be replayed against a device that has since
been released and reclaimed by someone else. `MkDeviceRelease` invalidates the
claim and its grants.

```
  driver capsule                     kernel broker
    |  MkDeviceClaim(device) -------->  record { owner pid, epoch }
    |  <----- claim id, epoch
    |  ... every grant below carries this claim and epoch ...
    |  MkDeviceRelease(device) ------>  drop the claim and all its grants
```

Claiming and releasing require the `Driver` capability; enumerating devices with
`MkDeviceList` requires `DeviceEnum`
([capabilities and tokens](../security/capabilities-and-tokens.md)).

---

## Interrupts

`MkIrqBind` (`src/syscall/microkernel/irq/bind.rs`) sets up interrupt delivery for
a claimed device. The broker core is at `src/hardware/broker/irq`.

For a line-based (INTx) interrupt:

```
  MkIrqBind(device, epoch, gsi, flags=0, vector_count=0, out)
    verify the device is claimed and the epoch is current
    allocate a broker vector from the pool (0x81..0xC0)
    program the IO-APIC route: gsi -> vector on this CPU
        ioapic::program_route_external(gsi, vector, dest_apic_id)   bind.rs:84
    mask the GSI                                                    bind.rs:88
    allocate a grant id, record { grant, pid, device, gsi, vector }
    write { grant_id, vector } to out
```

The line is masked at bind time and stays masked until the driver acknowledges,
so it cannot re-fire before the driver is ready. The driver then runs a poll and
ack loop:

```
  MkIrqPoll(grant)   read { seq, overflow }
                       seq advances once per delivered interrupt
                       overflow counts interrupts dropped while the driver was behind
  MkIrqAck(grant)    I have handled up to the last seq, unmask the line, accept the next
```

On the kernel side, the interrupt arriving on the allocated vector bumps the
grant's `seq` and masks the line; `MkIrqAck` unmasks it
(`src/hardware/broker/irq/release.rs`, `ack_grant` calls `ioapic::mask(gsi,
false)` for INTx). This is level-safe: the line is quiet between the interrupt and
the acknowledgement, so a level-triggered device does not storm the CPU.

`MkIrqWait` (tag MIRW) exists as a number but has no handler today; drivers use
the poll and ack loop. This is called out so nobody wires a driver against a wait
that never fires.

MSI-X follows the same grant model but allocates a contiguous block of vectors
from the pool and programs the device's MSI-X table instead of an IO-APIC line.
The driver derives per-vector grant ids as `grant + i` and vectors as `vector +
i`. The pool is contiguous precisely so an N-vector MSI-X request gets N
consecutive grants. All IRQ calls require the `Irq` capability, and the broker
additionally checks the caller owns the named grant.

---

## MMIO

`MkMmioMap` maps a claimed device's BAR into the caller's address space
(`src/hardware/broker/mmio`). The broker vets the requested region against the
device's PCI record, so a driver can only map a BAR that belongs to the device it
claimed, and maps that physical range into the capsule's page tables.
`MkMmioUnmap` removes the mapping. Both require the `Mmio` capability. This is how
a driver reaches its device registers without the kernel handing out arbitrary
physical memory.

## DMA

`MkDmaMap` pins a driver's buffer and returns a DMA address a device can read or
write (`src/hardware/broker/dma`). Pinning prevents the pages from being moved or
reclaimed while the device is using them, and the translation gives the device
the physical address behind the driver's virtual buffer. `MkDmaUnmap` releases
the pinning. Both require the `Dma` capability and device ownership. The kernel
holds the physical truth; the driver holds a buffer it allocated and a DMA
address the broker vetted.

## Port IO

Port IO is x86_64 only, and the PIO broker is compiled only for that target
(`src/hardware/broker`, gated by `cfg(target_arch = "x86_64")`). The aarch64 and
riscv64 backends have no port IO; their devices are reached entirely through
MMIO.

```
  MkPioGrant(device, ports)   grant a port range for the claimed device
  MkPioRead(port, width)      execute IN with width validation
  MkPioWrite(port, val, width)execute OUT with width validation
  MkPioRelease(grant)         release the port grant
```

The broker allocates ports from a global bitmap and validates that every read and
write is on a granted port and a legal width, so a driver cannot poke ports
outside its grant. All PIO calls require the `Pio` capability. The PS/2 input
driver is the canonical user: it grants the i8042 data and status ports and
drives the controller through `MkPioRead` and `MkPioWrite`.

---

## Why grants, not access

The whole broker exists so that a compromised or buggy driver capsule is bounded.
It can only touch the device it claimed, the BARs and ports the broker vetted, the
IRQ line it bound, and the buffers it pinned. It cannot reach another device,
arbitrary physical memory, or another capsule's interrupt. The capability bits
gate which kinds of grant a driver may ask for at all, and the per-grant ownership
check stops one driver from operating another's grant even if it holds the same
capability. The device claim and epoch tie the whole set together so grants
cannot outlive the claim that authorised them.

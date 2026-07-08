# IRQ Grants

A driver capsule does not handle interrupts in ring 0. Instead it binds its device's interrupt
through the broker, which programs the interrupt controller to deliver on a kernel-owned vector,
and then the capsule waits for and acknowledges interrupts through syscalls. The capsule never
touches the controller or the MSI-X table directly. This page documents `MkIrqBind` and its
companions across the three architectures. The code is under `src/hardware/broker/irq/`.

## The two bind modes

On x86 a bind is either legacy INTx or MSI-X, and both start from the same claim and epoch check
(`src/hardware/broker/irq/bind.rs:52`):

```
  bind(pid, req):
      claim = lookup(device_id); verify pid and epoch
      if req.flags & BIND_MSIX:  bind_msix(...)
      else:                      bind_intx(...)
```

**INTx** (`bind.rs:68`) validates the request against the device's interrupt pin and line,
allocates a broker vector slot, programs the IO-APIC to route the device's GSI to that vector on
the current LAPIC, masks the line, and records a single `Intx` grant. A GSI that is already
bound is refused with `AlreadyBound`, and if the IO-APIC program fails the just-allocated slot is
freed before returning, so no error path leaks a vector.

**MSI-X** (`bind.rs:105`) is the modern path. The kernel walks the device's MSI-X capability,
validates the table and PBA BARs against the claimed device, allocates `vector_count` contiguous
broker vectors, programs that many MSI-X table entries with the LAPIC redirect, enables MSI-X,
and unmasks each entry, recording one grant per vector. The defining property is stated in the
module doc: *the capsule never sees the table address and never writes to it; it only receives
the base grant id and base vector*. All hardware-touching steps go through a `MsixOps`
indirection so the path is testable, and the vectors are allocated contiguously so a failure
frees the whole run.

## Validation order

The MSI-X validator (`irq/validate.rs`) returns errors in a fixed priority so a capsule gets a
deterministic reason for a malformed request: unknown flags, then bad vector count (zero, larger
than the broker pool, or larger than the device's table), then no device handle, then no MSI-X
capability, then a bad table or PBA BAR, then a non-zero `irq_source` (MSI-X requires it be
zero), then already-bound (MSI-X bind is all-or-nothing per device per pid). The validators are
pure functions over plain inputs, no globals and no MMIO, run after the bind path has looked up
the kernel-side state.

## Waiting, polling, acknowledging

Once bound, a capsule receives interrupts through three operations (`irq/mod.rs`): `wait_arm` and
`wait_disarm` register interest so the capsule can block until its interrupt fires, `poll`
reports pending interrupts without blocking, and `ack_grant` acknowledges one so the next can be
delivered. The kernel-side interrupt entry for a broker vector was installed in the
[IDT](../interrupts/idt.md) from `arch::interrupt::broker`; when it fires, the dispatch path
(`irq/dispatch.rs`) records the pending interrupt against the grant and wakes a waiting capsule.
The capsule's handler thus runs in ring 3, driven by syscalls, never in an interrupt context.

## Multiple architectures

IRQ is the one grant class whose backend is genuinely architecture-specific, and the module
selects it by `target_arch` (`irq/mod.rs:22`):

```
  x86_64    IO-APIC redirection (INTx) + PCI MSI-X
  aarch64   GICv3 SPIs
  riscv64   PLIC external sources
```

The bind, poll, wait, ack, and release surface is the same across all three; only the controller
programming underneath differs. The shared request and grant types live in `irq/types.rs`, and
each backend implements the same operations against its platform's controller. Revocation,
`ack_grant`, `release_for_device`, and `release_all_for_pid`, unbinds the vector and, for MSI-X,
tears the table entry down; see [revocation](revocation.md).

## Source

```
  src/hardware/broker/irq/mod.rs        arch backend selection and the public surface
  src/hardware/broker/irq/bind.rs        INTx and MSI-X bind (x86)
  src/hardware/broker/irq/validate.rs    the pure MSI-X validators and error order
  src/hardware/broker/irq/dispatch.rs    interrupt delivery to the waiting capsule
  src/hardware/broker/irq/aarch64/       GICv3 backend
  src/hardware/broker/irq/riscv64/       PLIC backend
```

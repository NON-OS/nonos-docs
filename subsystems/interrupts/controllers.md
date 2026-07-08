# Interrupt Controllers

Between a device line and the CPU vector sits an interrupt controller. NØNOS supports two: the
legacy 8259 PIC, which it remaps out of the way of the CPU exception vectors, and the local
APIC, which is the preferred controller once it is up. This page documents both and the gate
that decides which one an acknowledgement goes to. The code is under `src/interrupts/pic/` and
`src/interrupts/apic/`.

## The 8259 PIC

The PIC delivers its lines starting at vector 8 by default, which collides with the CPU
exception vectors, so the first thing the kernel does is remap it. `pic::init`
(`src/interrupts/pic/init.rs:25`) runs the 8259 four-word initialization sequence on the
master and slave pair:

```
  save the current interrupt masks
  ICW1: begin init, expect ICW4          (master and slave)
  ICW2: master vector offset 0x20 (32)   slave offset 0x28 (40)
  ICW3: cascade wiring (slave on line 2)
  ICW4: 8086 mode
  restore the saved masks
```

After this the master's eight lines land on vectors 32 to 39 and the slave's on 40 to 47,
which is the legacy IRQ range the [IDT](idt.md) reserves; timer IRQ 0 becomes vector 32,
keyboard IRQ 1 becomes vector 33. The remap preserves the masks that were in place rather than
unmasking everything, and the module exposes `mask_irq` / `unmask_irq` and `mask_all` for
line-level control, and `send_eoi` to acknowledge a line.

## The local APIC

The local APIC is the modern per-CPU controller, and the interrupts module's APIC surface is a
thin façade over the system APIC driver: `apic::init` delegates to `sys::apic::init`
(`src/interrupts/apic/init.rs:17`), `apic::send_eoi` to `sys::apic::eoi`, and `apic::is_enabled`
reports whether the APIC came up. The APIC is where the [SMP](../smp/tlb-shootdown.md)
inter-processor interrupts and the LAPIC timer live; this module consumes it for the
end-of-interrupt path and leaves the bring-up to the driver.

## Choosing the controller

Every handler's acknowledgement goes through the same gate:

```
  if apic::is_enabled():  apic::send_eoi()
  else:                   pic::send_eoi(irq_line)
```

The kernel prefers the APIC and falls back to the PIC only when the APIC is not enabled. This
is the single decision that keeps the two controllers from disagreeing: an interrupt is
acknowledged to exactly one of them, chosen by whether the APIC is live, so a line is never
double-acknowledged or left hanging. In the normal boot the APIC comes up early and the PIC,
having been remapped to safe vectors, sits masked as a fallback rather than a participant.

## Source

```
  src/interrupts/pic/init.rs     the 8259 remap sequence
  src/interrupts/pic/mask.rs     per-line and global masking
  src/interrupts/pic/eoi.rs      the PIC end-of-interrupt
  src/interrupts/apic/           the façade over sys::apic (init, eoi, is_enabled)
```

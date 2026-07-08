# Platform Discovery

Every architecture has to learn its own machine: how many CPUs, where the interrupt controller is, what
devices exist. x86_64 learns this from ACPI tables; aarch64 and riscv64 learn it from a flattened device
tree. This page documents the two discovery paths and the arch-gated features that follow from them. The
code is `src/arch/fdt/`, `src/arch/x86_64/acpi/`, and the arch gating in `src/arch/mod.rs`.

## Two discovery models

The kernel supports the two dominant firmware-description models, selected by architecture:

```
  x86_64             ACPI tables      (src/arch/x86_64/acpi/)
  aarch64, riscv64   flattened device tree (FDT)   (src/arch/fdt/)
```

`src/arch/mod.rs:20` compiles the FDT module only for aarch64 and riscv64, and the ACPI module is part of
the x86_64 tree, so each build carries exactly the discovery mechanism its target uses. Both answer the
same questions, the CPU inventory the [SMP](../subsystems/smp/README.md) bring-up needs and the interrupt
topology the [interrupt](../subsystems/interrupts/README.md) layer needs, in the format the platform's
firmware provides.

## The device tree

The FDT module (`src/arch/fdt/mod.rs`) is a from-scratch flattened-device-tree parser: it handles the
big-endian FDT encoding, the header, the token stream, the string table, and property decoding, and
exposes a `Fdt`, a `Property` type, and find and walk helpers over the tree. This is how an ARM or RISC-V
build discovers its hardware: the bootloader passes a device-tree blob, and the kernel walks it to find
the memory ranges, the interrupt controller (the GIC or PLIC), the timer, and the devices. It is the
device-tree analogue of the x86 ACPI table walk.

## ACPI

The [x86_64 backend](x86_64.md) owns the ACPI side: the MADT for the interrupt and processor topology,
the HPET, the IO-APIC and LAPIC addresses, the interrupt source overrides, the NUMA regions, and the PCIe
segments. The two models are not mixed, an x86 build reads ACPI and an ARM or RISC-V build reads the
device tree, and the subsystems above them (SMP, interrupts, the PCI enumeration behind the
[hardware broker](../subsystems/hardware-broker/devices.md)) consume whichever one the target provides.

## Arch-gated features

Discovery is one place the architecture shows through; there are a few others where a feature simply does
not exist off one architecture, and the kernel gates them rather than emulating them:

- **PIO** (port-mapped I/O) is an x86 instruction class. The [hardware broker](../subsystems/hardware-broker/pio.md)
  compiles its PIO module only on x86_64 (`src/hardware/broker/mod.rs`), and the PIO syscalls fail closed
  with `ENOSYS` on other architectures.
- **The IRQ backend** differs per architecture: the broker's [IRQ grants](../subsystems/hardware-broker/irq.md)
  use the IO-APIC and MSI-X on x86, the GIC on ARM, and the PLIC on RISC-V, selected by `target_arch`.

These gates are the honest form of multi-architecture support: where a capability is genuinely
arch-specific, the kernel exposes it where it exists and fails cleanly where it does not, rather than
pretending every architecture is the same.

## Source

```
  src/arch/fdt/mod.rs          the flattened-device-tree parser (aarch64, riscv64)
  src/arch/x86_64/acpi/mod.rs   the ACPI tables (x86_64)
  src/arch/mod.rs               the FDT arch gating
  src/hardware/broker/mod.rs    the PIO arch gate
```

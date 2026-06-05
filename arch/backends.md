# Architecture Backends

This page describes the concrete backend layout in `src/arch`: the cfg-selected
`Arch` alias, the `ArchOps` trait, platform discovery, and backend status. Read
[Architecture Overview](../architecture/overview.md) first.

---

## 1. Module boundary

`src/arch/mod.rs` exports the generic arch modules, gates FDT to aarch64 and
riscv64, and gates x86_64 boot support to x86_64 (`src/arch/mod.rs:17`). It
selects the active backend with the `Arch` type alias: x86_64 maps to
`x86_64::abi::X86_64`, aarch64 maps to `aarch64::abi::Aarch64`, and riscv64
maps to `riscv64::abi::Riscv64` (`src/arch/mod.rs:46`).

```
  generic kernel code
        |
        | Arch as ArchOps
  +----------+   +----------+   +----------+
  | x86_64   |   | aarch64  |   | riscv64  |
  +----------+   +----------+   +----------+
```

## 2. ArchOps

`ArchOps` is the arch leaf trait. It defines halt, interrupt enable, interrupt
disable, interrupt state, current CPU id, time counter read, single address TLB
flush, and address space switch (`src/arch/abi.rs:36`).

| Method | Contract source |
|--------|-----------------|
| `halt` | `src/arch/abi.rs:37` |
| `enable_interrupts` | `src/arch/abi.rs:40` |
| `disable_interrupts` | `src/arch/abi.rs:48` |
| `interrupts_enabled` | `src/arch/abi.rs:56` |
| `current_cpu_id` | `src/arch/abi.rs:59` |
| `read_time_counter` | `src/arch/abi.rs:63` |
| `flush_tlb_one` | `src/arch/abi.rs:69` |
| `switch_address_space` | `src/arch/abi.rs:78` |

The current boundary is intentionally narrow. The trait documentation states
that IRQ vector allocation, MMIO, PIO, DMA grants, syscall entry, and per-arch
timer devices live behind later boundaries (`src/arch/abi.rs:25`).

## 3. x86_64 backend

The x86_64 backend implements `ArchOps` in `src/arch/x86_64/abi.rs`. It halts
with `cli; hlt`, enables interrupts with `sti`, disables with `cli`, reads IF
from RFLAGS bit 9, reads the APIC id from CPUID leaf 1, reads time through
`rdtsc`, flushes with `invlpg`, and switches address space by writing CR3
(`src/arch/x86_64/abi.rs:28`).

x86_64 also owns the ACPI module. The ACPI module exports MADT, HPET address,
interrupt overrides, IO-APIC data, LAPIC address, NMI configs, NUMA regions,
PCIe segments, processors, and table accessors (`src/arch/x86_64/acpi/mod.rs:33`).

## 4. aarch64 backend

The aarch64 backend exposes ABI, assembly, boot, context, CPU, exceptions, FPU,
GIC, MMU, PSCI, security, timer, and UART modules (`src/arch/aarch64/mod.rs:17`).
Its `ArchOps` implementation delegates each trait method to the matching
backend module: halt, IRQ enable and disable, IRQ state, CPU id, time counter,
TLB flush, and address space switch (`src/arch/aarch64/abi/archops.rs:24`).

## 5. riscv64 backend

The riscv64 backend exposes ABI, assembly, boot, context, CPU, FPU, interrupts,
MMU, PLIC, SBI, security, timer, and UART modules (`src/arch/riscv64/mod.rs:17`).
Its `ArchOps` implementation delegates the same eight trait methods to backend
modules (`src/arch/riscv64/abi/archops.rs:24`).

## 6. Platform discovery

FDT is compiled only for aarch64 and riscv64 at the arch module boundary
(`src/arch/mod.rs:20`). The FDT module exports endian, errors, find helpers,
headers, parser, properties, strings, tokens, and walker code, then re-exports
`FdtError`, `Fdt`, and `Property` (`src/arch/fdt/mod.rs:17`).

PIO is x86_64-only in the hardware broker. The broker gates its PIO module with
`#[cfg(target_arch = "x86_64")]`; non-x86 builds skip that broker submodule
(`src/hardware/broker/mod.rs:29`).

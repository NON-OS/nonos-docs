# Architecture

How NØNOS runs on more than one instruction set. Generic kernel code never names an architecture; it
calls a small trait, `ArchOps`, through one `cfg`-selected type alias, and the build links the backend
for its target. x86_64 is the production architecture; aarch64 and riscv64 are architecture-ready
backends that implement the same boundary. Read the [architecture overview](../architecture/overview.md)
first for the system-level picture.

| Page | What it covers |
|------|----------------|
| [boundary.md](boundary.md) | The `ArchOps` trait, its eight leaf primitives, the `Arch` alias, the fail-to-link discipline, and why the boundary is narrow. |
| [x86_64.md](x86_64.md) | The production backend: the direct instruction sequences and ACPI platform discovery. |
| [aarch64.md](aarch64.md) | The architecture-ready ARM backend: the GIC, PSCI, MMU, and generic timer, with an honest maturity note. |
| [riscv64.md](riscv64.md) | The architecture-ready RISC-V backend: the PLIC, SBI, MMU, and `mtime`, with the same maturity note. |
| [platform-discovery.md](platform-discovery.md) | ACPI versus the flattened device tree, and the arch-gated features (PIO, the IRQ backend). |

The principle that runs through the section is that portability is a boundary, not a sprinkling of
`cfg`. The shared kernel goes through `ArchOps` and the other arch seams (the syscall bridge, the IRQ
backend, PIO) rather than into per-arch modules directly, a backend that cannot implement a primitive
does not exist as an `Arch` and the build fails to link rather than misbehaving, and where a capability
is genuinely arch-specific the kernel exposes it where it exists and fails cleanly where it does not.
The maturity ladder is explicit: x86_64 in production, aarch64 and riscv64 architecture-ready, then
QEMU, then hardware.

## Sources

The boundary is `src/arch/abi.rs` and `src/arch/mod.rs`; the backends are `src/arch/x86_64/`,
`src/arch/aarch64/`, and `src/arch/riscv64/`, each with an `abi` module implementing `ArchOps`. Platform
discovery is `src/arch/x86_64/acpi/` and `src/arch/fdt/`. Every page is verified against those trees with
`file:line` references.

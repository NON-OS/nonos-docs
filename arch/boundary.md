# The Architecture Boundary

NØNOS runs on more than one instruction set, and it does so without scattering `cfg` branches through
the kernel. Generic code calls a small trait, `ArchOps`, through one type alias, and the build selects
which backend that alias resolves to. This page documents the boundary. The code is `src/arch/abi.rs`
and `src/arch/mod.rs`.

## One trait, one alias

The generic kernel never names an architecture. It calls the active backend through the `Arch` type
alias, which `src/arch/mod.rs:46` selects by target:

```
  #[cfg(target_arch = "x86_64")]  pub type Arch = x86_64::abi::X86_64;
  #[cfg(target_arch = "aarch64")] pub type Arch = aarch64::abi::Aarch64;
  #[cfg(target_arch = "riscv64")] pub type Arch = riscv64::abi::Riscv64;
```

`Arch` implements `ArchOps`, so shared code writes `Arch::halt()` or `Arch::current_cpu_id()` and the
compiler resolves it to the backend for the target being built. Adding an architecture is adding a
backend type that implements the trait and a `cfg` arm here; no generic code changes. This is the
concrete form of the project's multi-architecture discipline: the shared path goes through the trait,
never into a per-arch module directly.

## The eight leaf primitives

`ArchOps` (`src/arch/abi.rs:36`) is deliberately small, the eight primitives that genuinely cannot be
written portably:

```
  halt() -> !                          park the CPU forever
  enable_interrupts()   (unsafe)       unmask IRQs on this CPU
  disable_interrupts()  (unsafe)       mask IRQs on this CPU
  interrupts_enabled() -> bool         is the IRQ flag set
  current_cpu_id() -> u32              this CPU's stable id
  read_time_counter() -> u64           the monotonic per-CPU tick counter
  flush_tlb_one(addr)   (unsafe)       invalidate one TLB entry on this CPU
  switch_address_space(root) (unsafe)  install a page-table root on this CPU
```

Each is a single hardware operation with no portable expression: halting, the interrupt flag, the CPU
id, the cycle counter, a single-page TLB invalidation, and the page-table root switch. The unsafe ones
carry explicit contracts in the source, enable and disable must be paired or the CPU strands,
`flush_tlb_one` invalidates only the calling CPU (cross-CPU shootdown is the [SMP layer's](../subsystems/smp/tlb-shootdown.md)
job), and `switch_address_space` requires a valid root or it faults. `read_time_counter` returns
platform-defined units (TSC ticks, the generic timer, or `mtime`); callers that want wall-clock time go
through [`sys::clock`](../subsystems/time-and-clock/README.md) instead.

## Fail to link, not fail silently

The trait's design rule is stated in its own doc: the primitives are infallible, and an architecture
that cannot implement one yet must not have an `ArchOps` impl at all. The consequence is that a build
for an incomplete architecture fails to link rather than compiling and doing the wrong thing at runtime.
There is no default method that silently no-ops; a backend is complete or it does not exist as an
`Arch`. This is what lets the kernel treat `Arch::` calls as trustworthy on every target it actually
builds for.

## A narrow boundary on purpose

The boundary is intentionally small, and the source says why: this is the first phase, eight leaf
primitives, and the larger arch-specific concerns, IRQ vector allocation, MMIO, PIO, and DMA grants,
the syscall entry path, and the per-arch timer device, live behind their own boundaries rather than
being crammed into `ArchOps`. Some of those already exist as separate seams: the
[hardware broker](../subsystems/hardware-broker/README.md) gates PIO to x86_64, the
[syscall boundary](../subsystems/syscall/boundary.md) has its own arch bridge, and the
[interrupt](../subsystems/interrupts/README.md) controllers are arch-specific. `ArchOps` is the leaf
layer under all of them.

## Source

```
  src/arch/abi.rs    the ArchOps trait and its eight primitives
  src/arch/mod.rs    the cfg-selected Arch alias and the module gating
```

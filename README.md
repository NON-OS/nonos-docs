# NØNOS Documentation

Reference documentation for the NØNOS microkernel: a from-scratch, `no_std`,
capability-based, RAM-resident operating system. It is multi-architecture by
construction: x86_64 is the production-first target, with aarch64 and riscv64
as architecture-ready backends behind a shared arch boundary.

This repository is the single source of truth for how the system is built and
why. It is written against the actual source tree, with file and line
references, so a reader can open the kernel beside a page and confirm every
claim. Nothing here is aspirational. If a page describes behaviour, that
behaviour exists in the code at the cited location.

The documentation is mounted into the kernel repository as a Git submodule at
`docs/`, so a checkout of the kernel always carries the matching documentation
revision.

## Layout

```
nonos-docs/
├── architecture/      System model, boot, memory, scheduling, the whole map
├── abi/               Syscall numbers, calling convention, per-call contracts
├── security/          Trust anchor, capsule signing, capabilities, tokens
├── subsystems/        Per-subsystem deep dives (broker, input, graphics, ipc)
└── assets/            Diagrams and figures
```

## Start here

Read the architecture overview first. It is the map the rest of the
documentation hangs off.

| Page | What it covers |
|------|----------------|
| [the-nonos-path.md](the-nonos-path.md) | The single institutional and engineering document: what NØNOS is today, the microkernel and capsule model, the SDK and the GUI, the true path to build on both, the cryptographic spine, the NOX settlement layer, and the road to x402 and agentic payments on Base. |
| [architecture/overview.md](architecture/overview.md) | The full system in one document: the arch boundary, privilege model, boot order, memory, capsules, verified spawn, capabilities, syscalls, IPC, scheduling, the hardware broker, and the input and graphics paths. |

### Reference

| Section | Page | What it covers |
|---------|------|----------------|
| ABI | [abi/syscalls.md](abi/syscalls.md) | Every syscall: calling convention, numbers, capability, and semantics. |
| ABI | [abi/errors.md](abi/errors.md) | The error codes, the return convention, and what causes each. |
| Security | [security/capsules-and-trust.md](security/capsules-and-trust.md) | Capsule format, signing, the trust anchor, and the verified-spawn gate. |
| Security | [security/capabilities-and-tokens.md](security/capabilities-and-tokens.md) | The capability bits, their enforcement, and the capability token. |
| Subsystems | [subsystems/](subsystems/) | Per-subsystem deep dives: boot, memory, process model, ELF loader, scheduler, SMP, IPC, broker, interrupts, input, graphics, networking, storage, time, crypto. |
| Userland | [userland/](userland/) | The capsule userland model, the nonos_libc SDK, and the desktop stack. |
| Build | [build/](build/) | The toolchain, the cargo and Make build, and the capsule signing pipeline. |
| Architecture | [arch/](arch/) | The x86_64, aarch64, and riscv64 backends behind the arch boundary. |

Each page follows the same rule as the overview: code-accurate, cross
referenced, and verifiable against the tree.

## Conventions

Source references use the form `path/to/file.rs:LINE` relative to the kernel
repository root. Function and type names are quoted exactly as they appear in
the tree. Diagrams are plain ASCII so they render identically in a terminal, a
pager, a code review, and a browser.

## License

This documentation is part of the NØNOS project and is licensed under the GNU
Affero General Public License, version 3. The full text is in [LICENSE](LICENSE).
The documentation carries the same terms as the kernel it describes: the right
to use, study, share, and modify it, with the obligation to pass those same
rights on.

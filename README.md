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

| Page | What it covers |
|------|----------------|
| [architecture/overview.md](architecture/overview.md) | The full system: privilege model, boot order, memory, capsules, verified spawn, capabilities, syscalls, IPC, scheduling, the hardware broker, and the input and graphics paths. Read this first. |

The remaining sections are built out subsystem by subsystem. Each page follows
the same rule as the overview: code-accurate, cross-referenced, and verifiable.

## Conventions

Source references use the form `path/to/file.rs:LINE` relative to the kernel
repository root. Function and type names are quoted exactly as they appear in
the tree. Diagrams are plain ASCII so they render identically in a terminal, a
pager, a code review, and a browser.

## License

Documentation for the NØNOS project. The kernel is distributed under the GNU
Affero General Public License v3. This documentation is part of the same
project and carries the same terms.

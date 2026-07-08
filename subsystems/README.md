# Subsystems

Deep dives into one subsystem at a time. Each page takes a box from the
[architecture overview](../architecture/overview.md) and expands it with the full
data structures, control flow, and source references.

The overview already describes every subsystem below at the level needed to
understand the system. These pages go further: every field, every state
transition, every edge case the code handles. Each is verified against the
source.

| Page | Subsystem | Overview section |
|------|-----------|------------------|
| [boot/](boot/README.md) | Boot and init sequence | 4 |
| [memory/](memory/README.md) | Physical frames, paging, unified VM, heap, faults, hardening, usercopy, zeroization | 5 |
| [process/](process/README.md) | The PCB, the process table, lifecycle, context switch, the supervisor | 6, 11 |
| [elf-loader/](elf-loader/README.md) | Parsing, validation, and mapping of capsule ELFs | 6, 7 |
| [scheduler/](scheduler/README.md) | Priority selection, preemption, sleep and wake | 11 |
| [smp/](smp/README.md) | Per-CPU data, CPU identity, TLB shootdown | 11 |
| [syscall/](syscall/README.md) | The ring boundary, the tag numbers, the capability contract, the router | 6, 10 |
| [ipc/](ipc/README.md) | Inboxes, routing and permission, the message envelope and MAC, pipes | 10 |
| [hardware-broker/](hardware-broker/README.md) | Device claim, IRQ, MMIO, DMA, PIO grants | 12 |
| [interrupts/](interrupts/README.md) | IO-APIC routing, GSI ownership, vector pool | 12 |
| [input/](input/README.md) | The input ring and the driver to shell path | 13 |
| [graphics/](graphics/README.md) | Surfaces, sharing, presentation, vsync | 14 |
| [networking/](networking/README.md) | The L2 to sockets network capsule stack | 9 |
| [storage/](storage/README.md) | Block drivers, ramfs, and the vfs capsules | 9 |
| [time-and-clock/](time-and-clock/README.md) | TSC calibration, the two time bases, entropy | 11 |
| [crypto/](crypto/README.md) | The in-tree crypto stack and what uses each primitive | 15 |
| [proof-system/](proof-system/README.md) | The transparent STARK, the Poseidon-Goldilocks hash, the AIR catalog, and the Pedersen attestation | 15 |

The overview-section column points back to the matching section of the
architecture overview for the short version of each subsystem.

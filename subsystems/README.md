# Subsystems

Deep dives into one subsystem at a time. Each page takes a box from the
[architecture overview](../architecture/overview.md) and expands it with the full
data structures, control flow, and source references.

The overview already describes every subsystem below at the level needed to
understand the system. These pages go further: every field, every state
transition, every edge case the code handles. They are being written one at a
time, each verified against the source before it lands.

| Page | Subsystem | Overview section |
|------|-----------|------------------|
| boot.md | Boot and init sequence | 4 |
| memory.md | Physical memory, paging, unified VM | 5 |
| scheduler.md | Priority scheduling, preemption, sleep and wake | 11 |
| ipc.md | Endpoints, inboxes, call and reply | 10 |
| hardware-broker.md | Device claim, IRQ, MMIO, DMA, PIO grants | 12 |
| interrupts.md | IO-APIC routing, GSI ownership, vector pool | 12 |
| input.md | The input ring and the driver to shell path | 13 |
| graphics.md | Surfaces, sharing, presentation, vsync | 14 |
| crypto.md | The in-tree crypto stack and what uses each primitive | 15 |

Until a page lands, its subsystem is covered by the overview section listed
above. The numbers are stable references into that document.

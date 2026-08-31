# Input

How a keystroke or a pointer motion reaches an application. NØNOS keeps input in one place in the
kernel, a bounded ring, and moves the policy of routing events to windows out into a capsule.
Driver capsules post events into the ring, the kernel wakes a single router capsule, and the router
fans each event out to the consumer that owns the focus.

| Page | What it covers |
|------|----------------|
| [ring.md](ring.md) | The bounded MPSC ring, drop-on-full with a counter, the monotonic sequence number, and the single-waiter wakeup. |
| [path.md](path.md) | The `InputEvent` record, the `MkInputEventPost` / `Drain` / `Wait` syscalls, and the driver-to-router-to-consumer path. |
| [drivers.md](drivers.md) | The four input driver capsules (PS/2, i2c-PCI, i2c-HID, USB-HID), the claim / grant / IRQ / read / post model each follows, and how real hardware splits input across all of them at once. |

The shape to keep in mind is that the kernel does the least it can: it holds a 1024-entry ring,
counts drops rather than blocking a producer, publishes a sequence number so the consumer can wait
on an edge, and wakes exactly one router capsule. Everything above that, which window has focus,
how a scancode becomes a character, how events are delivered to a shell or a GUI, is a capsule's
job, reached over [IPC](../ipc/README.md). This mirrors the [hardware broker](../hardware-broker/README.md):
the kernel owns the minimal shared mechanism, capsules own the policy.

## Sources

The ring is `src/kernel_core/surface_registry/input_ring/` (split into `ring.rs`, `post.rs`,
`drain.rs`, `seq.rs`, `arm_waiter.rs`, `clear_waiter.rs`) with its types in
`src/kernel_core/surface_registry/types.rs`; the syscalls are
`src/syscall/dispatch/router/input_ops/` (`handle.rs`, `do_post.rs`, `do_drain.rs`, `do_wait.rs`);
and the router capsule is
`src/userspace/capsule_input_router/`. Driver capsules that post live under `src/hardware/` and
`src/userspace/`. Every page is verified against those trees with `file:line` references.

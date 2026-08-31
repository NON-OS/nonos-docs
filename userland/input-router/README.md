# The Input Router Capsule

`capsule_input_router` is the single consumer of the kernel input ring and the fan-out point for the
whole desktop. Driver capsules post hardware events into the kernel ring; this capsule drains that ring,
decides where each event belongs, and delivers it to the owning window over IPC. It is the userland
counterpart to the kernel [input subsystem](../../subsystems/input/README.md), and the routing decisions
here are the userland half of the [event path](../../subsystems/input/path.md).

Its source is organized into six top-level modules, and this documentation mirrors that structure so a
page can be read beside the folders it describes. The one fact to hold before anything else: the router
sees every keystroke and every pointer motion on the machine, and it is the only capsule that does. It
holds `InputSource`, the privileged consumer authority for the raw-input ring, so it can drain the ring
and speak IPC, but it cannot inject a synthetic event back into the ring the way a driver can. Drivers can
POST events but not DRAIN them; the router drains but cannot POST. That split is the spine of every page here.

## Identity

Everything the kernel and the service registry need to name and reach the router comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `input-router` | `Capsule.mk:5` |
| Service handle | `input_router` | `Capsule.mk:6`, `src/userspace/capsule_input_router/spawn.rs:31` |
| Namespace | `systems.nonos.input_router` | `Capsule.mk:11` |
| Service endpoint | `service:4320:input_router` | `Capsule.mk:12`, `spawn.rs:32` |
| Reply endpoint | `reply:4321:endpoint.input_router.reply` | `Capsule.mk:13`, `spawn.rs:33`, `spawn.rs:34` |
| Capability mask | `0x200019` | `Capsule.mk:16` |
| Binary name | `input_router` | `Capsule.mk:9` |
| Kernel mirror | `src/userspace/capsule_input_router` | `Capsule.mk:16` |

The mask `0x200019` decomposes into exactly four bits, checked against `src/capabilities/types/defs.rs`:

| Bit | Value | Grants |
|---|---|---|
| CoreExec | `0x000001` | run as a process |
| IPC | `0x000008` | send and receive on its endpoints |
| Memory | `0x000010` | map its own heap and stack |
| InputSource | `0x200000` | drain and wait on the global raw-input ring |

```
  0x200019 = 0x000001 + 0x000008 + 0x000010 + 0x200000
           = CoreExec + IPC + Memory + InputSource
```

The router holds `InputSource` (`0x200000`) because draining the raw-input ring is
now a privileged consumer operation, not an ordinary IPC call. The consumer gate
`can_input_consumer()` backs `MkInputEventDrain` and `MkInputEventWait`
(`src/syscall/contract/cap_table/mk.rs:106`), and it deliberately requires
`InputSource` while excluding `Irq`. The reason is spelled out at the gate: the
ring carries every keystroke, and the looser `can_input_source()` accepts `Irq`,
which every device driver holds, so a driver capsule could have drained the ring
and stolen the keystroke stream (`src/syscall/contract/cap_table/mk.rs:100`). The
tightened gate keeps drivers able to POST events (`MkInputEventPost`, still gated
on `can_input_source()`, `mk.rs:99`) but not DRAIN them, so the router is the one
capsule that can read the stream. It still cannot inject a synthetic event and it
holds no network, filesystem, graphics, or hardware capability at all. Compromising
the router yields those four bits and the right to ask the window manager,
compositor, and policy services a question.

## The six pillars

The source under `userland/capsule_input_router/src/` is six top-level modules, and this documentation is
one page per pillar (two of the modules pair with a partner). Data flows from the kernel ring on the left,
through classification and delivery, to the consumer on the right; the clients on the bottom answer the
questions each decision needs.

```
  sources/  ->  route/          ->  consumers
  drain the     classify and         (windows, shell,
  kernel ring   deliver each event    grab holders)
                    |
  server/  <-- IPC in (subscribe / grab / release / health)
  the loop and handlers, protocol/ the wire formats
                    |
  state/    the routing memory (cursor, grabs, subs, press, hover, key targets)
  clients/  the questions out (wm focus and hit-test, compositor, policy)
```

| Page | Mirrors | What it covers |
|---|---|---|
| [operations.md](operations.md) | `src/protocol/`, `src/server/` | The `NIRS` request frame and the `NINP` delivery frame, the four opcodes, the non-blocking IPC drain, the four handlers, and the reply path on port 4321. |
| [routing.md](routing.md) | `src/sources/`, `src/route/` | The kernel-ring batch drain and the routing decision engine: the grab-first / pointer / keyboard / broadcast order, and the pointer specialization with focus query, hit-test, and the press-drag latch. |
| [state.md](state.md) | `src/state/` | The routing memory: the cursor, the grab table, the subscription table, the per-key targets, the press and hover caches, and the `Context` that owns them all. |
| [clients.md](clients.md) | `src/clients/` | The outbound service clients: the window manager (focus, hit-test, route-focus), the compositor (display size, cursor update), the policy field read, and the shared `NIRS` wire helper. |
| [contributing.md](contributing.md) | the whole tree | Where to work, how to add an operation or a routing rule, the build and sign steps, and the code standards. |
| [debugging.md](debugging.md) | runtime | The boot marker, the first-event bench markers, and where to look when input never reaches a window or a grab is stuck. |

## Lifecycle

The router is spawned first in the desktop GUI fleet, before the compositor, so the rest of the desktop
comes up behind it (`src/userspace/init/spawn_plan/desktop_fleet/mod.rs`, `:72`). The spawn is idempotent
through an `is_alive` guard, verifies the embedded ELF, cert, manifest, and attestation, and registers
`input_router` on port 4320 (`src/userspace/capsule_input_router/spawn.rs:37`). It also runs as part of
the input-probe fleet (`src/userspace/init/spawn_plan/input_probe_fleet.rs:24`).

`_start` initializes the heap and calls `server::run`, which never returns (`src/main.rs:32`). Each loop
iteration drains pending IPC, does periodic maintenance every 64th tick, drains a batch of input events
from the kernel ring, routes each event, pushes a cursor update to the compositor if the cursor moved, and
blocks on the ring's sequence number when there is nothing left to do (`src/server/runner.rs:38`). So the
router never spins when idle and never blocks while events are pending. On a successful spawn the kernel
logs `[INPUT-ROUTER] capsule spawned`; the [debugging](debugging.md) page covers what each later marker
means.

## Source map

```
  Capsule.mk                                 slug, handle, ports, capability mask, kernel mirror
  userland/capsule_input_router/src/main.rs  _start -> heap_init -> server::run
  userland/capsule_input_router/src/protocol/  the NIRS request and NINP delivery wire formats
  userland/capsule_input_router/src/server/    the loop, the IPC drain, and the four op handlers
  userland/capsule_input_router/src/sources/   the kernel-ring batch drain
  userland/capsule_input_router/src/route/     the routing decision engine
  userland/capsule_input_router/src/state/     the routing memory
  userland/capsule_input_router/src/clients/   the outbound service clients
  src/capabilities/types/defs.rs                  the CoreExec / IPC / Memory / InputSource capability bits
  src/syscall/contract/cap_table/mk.rs       the per-syscall capability gate
  src/userspace/capsule_input_router/        the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/             the desktop and input-probe fleet spawn entries
```

Every reference above is verified against those trees.
</content>
</invoke>

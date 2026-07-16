# capsule_power

`capsule_power` is the userland power service: the one capsule that can reset or power off the machine.
It is deliberately tiny. It takes a request on a fixed IPC port, records a timestamp, and issues the
privileged kernel admin syscall that does the real work. The interesting detail is the ordering: reboot
builds its reply before it resets the machine, shutdown returns the syscall's result as the status. The
source is `userland/capsule_power/`.

Two things about this capsule are worth stating up front, because they shape everything below. First, it
is built into the image but is not spawned by the kernel at boot; it is a defined-but-not-spawned service
(see [Identity](#identity)). Second, reboot works but shutdown does not yet: the kernel's shutdown syscall
returns `E_NOTSUP` because NONOS has no AML interpreter to evaluate the DSDT `_S5` method
(`src/arch/x86_64/acpi/power_sleep.rs:34`, `src/syscall/dispatch/router/admin/shutdown.rs:19`).

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## Overview

The power capsule is a request-reply IPC server with three operations and no user interface, no window,
and no state beyond two timestamps. It exists so that the authority to reset or power off the machine
lives behind exactly one service, holding exactly one privileged capability, rather than being scattered
across the fleet. Any capsule that wants the machine to reboot sends a frame to the power service; the
power capsule is the only holder of the Admin capability that the reboot and shutdown syscalls require
(`Capsule.mk:13`, `src/syscall/contract/cap_table/admin.rs:22`).

`_start` initializes the heap and calls `server::run`, which loops on the fixed service port, parses each
frame, dispatches to a per-op handler, and replies to the attested sender
(`src/main.rs:33`, `src/server/runner.rs:28`). The two power operations are thin wrappers over the kernel
admin syscalls `mk_admin_reboot` and `mk_admin_shutdown`; the capsule itself touches no hardware and holds
no ACPI knowledge (`src/server/handlers/reboot.rs:29`, `src/server/handlers/shutdown.rs:28`).

## Identity

Everything the kernel and the service registry need to name and reach the power capsule comes from its
`Capsule.mk`.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `power` | `Capsule.mk:1` |
| Service handle | `power` | `Capsule.mk:2` |
| Namespace | `systems.nonos.power` | `Capsule.mk:7` |
| Service endpoint | `service:4448:power` | `Capsule.mk:8` |
| Reply endpoint | `reply:4449:endpoint.power.reply` | `Capsule.mk:9` |
| Capability mask | `0x219` | `Capsule.mk:13` |
| Binary name | `power` | `Capsule.mk:5` |
| Kernel mirror (declared) | `src/userspace/capsule_power` | `Capsule.mk:14` |

The service port the running capsule actually listens on is `4448`, hardcoded in the runner, which matches
the manifest endpoint (`src/server/runner.rs:25`). The reply endpoint declares port `4449`, but the runner
does not bind a reply port of its own; it replies directly to the attested sender pid returned by the
receive (`src/server/runner.rs:47`).

The mask `0x219` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x0001  CoreExec   bit()   1    types.rs:56
  0x0008  IPC        bit()   8    types.rs:59
  0x0010  Memory     bit()  16    types.rs:60
  0x0200  Admin      bit() 512    types.rs:65
  ------
  0x0219  = 1 + 8 + 16 + 512
```

The comment in `Capsule.mk` states the same decomposition and the reason for the shape: Admin gates
`AdminReboot` and `AdminShutdown`, and Debug is deliberately omitted so a power transition can never leak
to the serial surface (`Capsule.mk:10`). There is no Network, no FileSystem, no Crypto, no Graphics, and no
hardware, driver, or DMA bit. This is a reset button: one privileged verb and nothing else.

Spawned at boot? No. The Makefile includes `userland/capsule_power/Capsule.mk`, so the capsule is built,
signed, and part of the image artifact set (`Makefile:683`, `Makefile:882`). But no entry in the kernel's
init spawn plan spawns it: a search of `src/userspace/init/spawn_plan/` for the power slug or its port 4448
finds nothing, and unlike a spawned app such as the terminal there is no `src/userspace/capsule_power/`
kernel mirror in the tree even though `Capsule.mk:14` declares one. So there is no `[POWER] capsule
spawned` line in the boot fleet; the capsule is defined and built but launched on demand, and the test that
it is up is whether a service lookup for `power` resolves, not a boot line.

## Operation reference

Three operations, defined as `u16` opcodes (`src/protocol/ops.rs:17`):

| Op | Opcode | What it does | Kernel path | Handler |
|---|---|---|---|---|
| `OP_HEALTHCHECK` | `0x0001` | liveness ping; replies status `0` | none | `src/server/handlers/health.rs:20` |
| `OP_REBOOT` | `0x0002` | reset the machine | `mk_admin_reboot` -> `AdminReboot` -> `power_reboot::reboot` | `src/server/handlers/reboot.rs:23` |
| `OP_SHUTDOWN` | `0x0003` | power off the machine | `mk_admin_shutdown` -> `AdminShutdown` -> `power_sleep::shutdown` | `src/server/handlers/shutdown.rs:23` |

Every reply is a fixed 24-byte frame: the 20-byte header echoed back plus a 4-byte little-endian status
word (`src/server/respond.rs:19`, `src/protocol/header.rs:19`). An unknown opcode is answered with status
`E_BAD_OP` (`-38`) rather than dropped (`src/server/handlers/router.rs:31`, `src/protocol/errno.rs:17`).

The two power operations differ in a small but deliberate way.

`reboot` records the request time, builds the success reply, and only then calls `mk_admin_reboot`
(`src/server/handlers/reboot.rs:24`):

```
  reboot(state, out, req):
      now = mk_time_millis()
      if now > 0: state.last_reboot_request_unix = now   // reboot.rs:24
      n = respond::status(out, req, 0)                    // build the reply FIRST  reboot.rs:28
      mk_admin_reboot()                                   // then reset the machine reboot.rs:29
      return n
```

The reply is prepared before the reset because after `mk_admin_reboot` the machine restarts and the reply
would never be delivered; a caller should not wait on a reboot ack regardless.

`shutdown` instead calls `mk_admin_shutdown` and returns the syscall's result as the status
(`src/server/handlers/shutdown.rs:28`):

```
  shutdown(state, out, req):
      now = mk_time_millis()
      if now > 0: state.last_shutdown_request_unix = now  // shutdown.rs:26
      rc = mk_admin_shutdown()                             // shutdown.rs:28
      status = if rc == 0 { 0 } else { rc }               // shutdown.rs:29
      return respond::status(out, req, status)            // shutdown.rs:30
```

Today `mk_admin_shutdown` always returns `E_NOTSUP` (`-95`), because the kernel handler refuses before any
side-effecting register write: NONOS has no AML interpreter to evaluate the DSDT `_S5` object and read the
`SLP_TYPa` value, so shutdown reports the gap honestly instead of writing a meaningless PM1 register
(`src/syscall/dispatch/router/admin/shutdown.rs:19`, `src/arch/x86_64/acpi/power_sleep.rs:34`). A caller
that sends `OP_SHUTDOWN` therefore gets a `-95` status back and the machine stays up. Reboot, by contrast,
is real on any x86 box.

The timestamp is only recorded when the clock read is positive, so a failed `mk_time_millis` leaves the
last value in place rather than clobbering it with a zero or a negative
(`src/server/handlers/reboot.rs:25`, `src/server/handlers/shutdown.rs:25`).

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. Its three modules are `protocol` (the wire format), `server` (the loop
and the handlers), and `state` (the two timestamps) (`src/main.rs:22`).

`_start` initializes the heap and, on failure, exits with code `1` rather than running with no allocator;
on success it hands control to `server::run`, which never returns (`src/main.rs:29`).

The request loop is a fixed-port server (`src/server/runner.rs:28`):

1. Allocate a receive and a send buffer of `IPC_PAYLOAD_MAX` (256) bytes each and construct a fresh
   `PowerState` (`src/server/runner.rs:29`, `src/protocol/mod.rs:29`).
2. Block on `mk_ipc_recv_from(4448, ...)`, which returns the byte count and the sender pid; a receive of
   zero or fewer bytes or a sender pid of `0` is treated as spurious, so the loop yields and retries rather
   than replying to nobody (`src/server/runner.rs:34`, `:41`).
3. Route the received bytes through `route`, which parses the header, dispatches on the opcode, and returns
   the number of reply bytes to send (`src/server/handlers/router.rs:22`).
4. If the router produced a non-empty reply, send it to the attested sender pid with `mk_ipc_reply`
   (`src/server/runner.rs:47`).

The state is a `PowerState` holding `last_reboot_request_unix` and `last_shutdown_request_unix`, both
`u64`, both initialised to zero by a `const fn new` (`src/state/mod.rs:17`). Nothing reads them back; they
are an in-memory audit breadcrumb for the current process lifetime only and do not persist across a reboot.

## Protocol and IPC

The wire format is a fixed 20-byte header (magic `0x504F5752`, ASCII `POWR`, version 1) followed by a
typed payload, capped at 256 bytes total (`src/protocol/header.rs:17`, `src/protocol/mod.rs:29`). The
header layout, little-endian throughout (`src/protocol/decode.rs:19`, `src/protocol/encode.rs:19`):

```
  offset 0   u32   magic       0x504F5752 ("POWR")
  offset 4   u16   version     1
  offset 6   u16   op          OP_HEALTHCHECK | OP_REBOOT | OP_SHUTDOWN
  offset 8   u16   flags       echoed into the reply
  offset 10  u16   reserved    zeroed in replies
  offset 12  u32   request_id  echoed into the reply
  offset 16  u32   payload_len bytes of payload following the header
  offset 20  ..    payload     (unused by every current op)
```

`parse` validates length, then magic, then version, and rejects a frame whose declared `payload_len`
overruns the buffer; each failure returns a best-effort `Request` and a negative status so the server can
still reply with a structured error rather than dropping the frame
(`src/protocol/decode.rs:20`). The error statuses (`src/protocol/errno.rs:17`):

```
  E_BAD_OP       -38   opcode not one of the three known ops
  E_BAD_MAGIC    -71   first four bytes are not "POWR"
  E_BAD_LEN      -90   frame shorter than the header, or payload overruns
  E_BAD_VERSION  -93   version field is not 1
```

Every reply is header-plus-status: `response_header` echoes magic, version, op, flags, and request_id,
zeroes the reserved field, and sets `payload_len = 4`; `write_status` writes the 4-byte status at offset
20, for a 24-byte reply (`src/server/respond.rs:19`, `src/protocol/encode.rs:19`).

The two power ops reach the kernel through the userland libc admin wrappers, which issue raw syscalls by
their four-byte tag:

- `mk_admin_reboot` calls syscall `ARBT` (`AdminReboot`) (`userland/libc/src/admin/reboot.rs:19`,
  `userland/libc/src/syscall/numbers/admin.rs:18`).
- `mk_admin_shutdown` calls syscall `ASDN` (`AdminShutdown`) (`userland/libc/src/admin/shutdown.rs:19`,
  `userland/libc/src/syscall/numbers/admin.rs:19`).

On the kernel side the admin router matches these two tags plus `AdminPolicyPush`, checks the Admin
capability, and dispatches (`src/syscall/dispatch/router/admin/route.rs:25`, `:42`). `AdminReboot` calls
`power_reboot::reboot()` and returns `E_OK` (`src/syscall/dispatch/router/admin/reboot.rs:21`);
`AdminShutdown` returns `E_NOTSUP` without side effects
(`src/syscall/dispatch/router/admin/shutdown.rs:19`). Both are marked `audit_required` in the syscall
result (`src/syscall/dispatch/router/admin/route.rs:47`).

The kernel reboot is a real three-stage fallback that lands on any x86 board
(`src/arch/x86_64/acpi/power_reboot.rs:22`): it first tries the ACPI reset register (SystemIo or
SystemMemory) if the parsed tables provide one, then falls back to the 8042 keyboard-controller reset
(port `0x64`, value `0xFE`), and finally, if the machine is still alive, loads a null IDT and executes
`int3` to force a triple fault. The `mk_time_millis` wrapper the handlers use for the audit timestamp is
the `MkTimeMillis` syscall (`userland/libc/src/time/wall.rs:19`).

## Security analysis

Powering off a machine is a high-impact operation, so the authority to do it is concentrated deliberately.
The privileged step, actually resetting the machine, is a kernel syscall gated on the Admin capability:
`AdminReboot` and `AdminShutdown` are only honored when the calling capsule's token grants Admin and is
valid (`src/syscall/contract/cap_table/admin.rs:22`, `src/syscall/caps/checks/system.rs:25`). The power
capsule holds Admin in its `0x219` mask, and by design no other capsule in the fleet does, so a compromise
of any other capsule cannot reboot or shut down the box without first going through this capsule's IPC
(`Capsule.mk:13`).

Within the mask, Admin is the only power beyond the service baseline. CoreExec, IPC, and Memory are what
any request-reply server needs: run code, receive and reply on a port, and hold a bounded reply buffer.
Admin is the whole reason the capsule exists. It holds no Crypto, no FileSystem, no Network, no Graphics,
and no hardware capability, so the capsule that can reset the machine cannot read a key, write a file, open
a socket, draw to the screen, or touch a device register. That is the correct shape for a reset button.
This is structurally the same posture as the [policy](policy.md) capsule, which also holds exactly one
Admin-class power and is otherwise minimal; the [admin syscall family](../../subsystems/syscall/router.md)
is where the real work happens.

The honest boundary, restated in the [gaps](#honest-gaps), is that the gate is the Admin capability at the
kernel syscall, not a check inside this capsule. The handler does not attest the caller: any capsule that
can reach port 4448 can send `OP_REBOOT`, and the request will succeed if and only if the power capsule
itself holds Admin, which it does. In other words the real gate on who may power the machine is the
capability to reach the power service combined with the power capsule's own Admin grant, not per-caller
logic in the handler. There is no debounce and no rate limit, so repeated `OP_REBOOT` frames all attempt
to fire. Debug is deliberately absent from the mask, so a power transition never emits a serial line
(`Capsule.mk:12`).

## How to contribute

The source lives at `userland/capsule_power/`. The wire protocol is under `src/protocol/`, the server loop
and per-op handlers under `src/server/`, and the two-timestamp state under `src/state/`.

To add or change an operation:

1. Add the opcode constant to `src/protocol/ops.rs:17`, following the existing `u16` values.
2. Write the handler as a module under `src/server/handlers/` exposing a
   `pub fn run(state: &mut PowerState, out: &mut [u8], req: &Request) -> usize` that builds its reply
   through `respond::status` and returns the reply length, the way `reboot.rs` and `shutdown.rs` do. A
   health-style handler that needs no state takes `(out, req)` instead (`src/server/handlers/health.rs:20`).
3. Re-export it from `src/server/handlers/mod.rs:17` and add a match arm keyed on the new opcode in
   `src/server/handlers/router.rs:27`.

To build, sign, and verify the capsule, use the generated per-slug make targets, produced by the rule
template in `nonos-mk/capsule.mk:156` and instantiated because the Makefile includes this capsule's
`Capsule.mk` (`Makefile:683`):

```
  make nonos-mk-power                build the capsule ELF
  make nonos-mk-power-sign           produce the id cert, manifest, and attestation trailer
  make nonos-mk-power-verify         verify the signed artifacts against the trust anchor
  make nonos-mk-check-power-keys     check the per-capsule signing keys exist
```

There is no `nonos-mk-power-prod` target: unlike a desktop app such as the terminal, the power capsule has
no dedicated production-image profile, because it is not part of a spawned desktop fleet. It is pulled into
the image through the shared artifact and verify sets instead (`Makefile:731`, `Makefile:882`,
`Makefile:1087`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns a status word, never a panic; the release profile is
`panic = "abort"`, `Cargo.toml:27`); modular files, one unit per file, with `mod.rs` used only for
re-exports (`src/protocol/mod.rs:17`); and the AGPL header at the top of every source file, matching the
header on every existing module.

## Debugging

The power capsule emits no serial output by design, because Debug is not in its mask, so there is no boot
marker and no per-request log to grep for (`Capsule.mk:12`). Confirming it is reachable means confirming a
service lookup for `power` resolves, not reading a boot line; recall from [Identity](#identity) that this
capsule is built into the image but is not spawned by the init spawn plan.

Failure modes and where to look:

- A reboot that appears to hang is by design. `reboot` builds its success reply before calling
  `mk_admin_reboot`, and the machine resets before the reply can be delivered, so a caller must not wait on
  a reboot ack (`src/server/handlers/reboot.rs:28`). In QEMU the reboot path takes effect within
  milliseconds of any capsule sending `OP_REBOOT`.
- A shutdown that comes back with status `-95` is not a bug in this capsule. `shutdown` returns the admin
  syscall's `rc`, and the kernel currently returns `E_NOTSUP` because there is no AML interpreter to read
  `_S5` (`src/server/handlers/shutdown.rs:29`, `src/arch/x86_64/acpi/power_sleep.rs:34`). The machine
  correctly stays up.
- A structured error status other than a power result points at the frame, not the machine. `-71`
  (`E_BAD_MAGIC`) means the first four bytes were not `POWR`, `-93` (`E_BAD_VERSION`) means the version
  field was not 1, `-90` (`E_BAD_LEN`) means the frame was short or the payload length overran, and `-38`
  (`E_BAD_OP`) means the opcode was none of the three (`src/protocol/decode.rs:20`,
  `src/server/handlers/router.rs:31`).
- No reply at all means the receive was treated as spurious: a byte count of zero or fewer, or a sender pid
  of `0`, makes the loop yield and retry without replying (`src/server/runner.rs:41`).

## Honest gaps

Stated plainly:

- Shutdown does not work yet. `OP_SHUTDOWN` returns `E_NOTSUP` (`-95`) because the kernel has no AML
  interpreter to evaluate the DSDT `_S5` method; the entry refuses before any register write rather than
  writing a meaningless `SLP_TYP` (`src/arch/x86_64/acpi/power_sleep.rs:34`). Reboot works.
- There is no caller attestation in the handler, so any capsule that can reach port 4448 can request a
  reboot; the real gate is the Admin capability at the kernel syscall plus the power capsule's own Admin
  grant, not logic here.
- There is no debounce or rate limit, so repeated requests all fire.
- The recorded timestamps are never read back and do not persist across a reboot.
- The capsule declares a kernel mirror at `src/userspace/capsule_power` (`Capsule.mk:14`), but that path
  does not exist in the tree and no spawn-plan entry launches the capsule, so it is defined-but-not-spawned
  at boot.

## Source map

```
  userland/capsule_power/src/main.rs                        _start -> heap init -> server::run
  userland/capsule_power/src/protocol/header.rs             magic POWR, version, 20-byte header, Request
  userland/capsule_power/src/protocol/{decode,encode}.rs    parse the header, build the reply header/status
  userland/capsule_power/src/protocol/ops.rs                OP_HEALTHCHECK | OP_REBOOT | OP_SHUTDOWN
  userland/capsule_power/src/protocol/errno.rs              E_BAD_OP/MAGIC/LEN/VERSION
  userland/capsule_power/src/server/runner.rs               the loop, recv-from and reply-to-sender on 4448
  userland/capsule_power/src/server/handlers/router.rs      opcode dispatch
  userland/capsule_power/src/server/handlers/reboot.rs      reply-then-reset
  userland/capsule_power/src/server/handlers/shutdown.rs    shutdown syscall, rc as status
  userland/capsule_power/src/server/handlers/health.rs      liveness ping
  userland/capsule_power/src/server/respond.rs              header + status reply builder
  userland/capsule_power/src/state/mod.rs                   the two request timestamps
  userland/capsule_power/Capsule.mk                         slug, handle, ports, 0x219 mask, kernel mirror
  userland/libc/src/admin/{reboot,shutdown}.rs              mk_admin_reboot / mk_admin_shutdown wrappers
  src/syscall/dispatch/router/admin/                        AdminReboot / AdminShutdown kernel dispatch
  src/syscall/contract/cap_table/admin.rs                   Admin capability gate on the admin syscalls
  src/arch/x86_64/acpi/power_reboot.rs                      real reset: ACPI reg, 8042, triple fault
  src/arch/x86_64/acpi/power_sleep.rs                       shutdown: E_NOTSUP until an AML evaluator lands
  src/capabilities/types.rs                                 the capability bit values
  nonos-mk/capsule.mk                                       the generated nonos-mk-power[-sign|-verify] targets
```

Every reference above is verified against those trees.

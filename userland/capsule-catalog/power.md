# capsule_power

`capsule_power` performs system reboot and shutdown. It is one of the smallest capsules in the tree: it
takes a request, records a timestamp, and issues the privileged kernel admin syscall. The interesting
detail is the ordering, reboot replies before it resets the machine, shutdown replies with the syscall's
result. Service `power` on port 4448, capability mask `0x219`. The source is `userland/capsule_power/`.

## Contents

- [The server loop](#the-server-loop)
- [The operations](#the-operations)
- [Reboot and shutdown](#reboot-and-shutdown)
- [State](#state)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop

`main.rs:28` initializes the heap and calls `server::run` (`src/server/runner.rs:28`), a request loop on
the fixed service port 4448 that replies directly to the sender. The frame is magic `0x504F5752`
("POWR"), version 1, a 20-byte header.

## The operations

Three operations (`src/protocol/ops.rs:17`):

```
  1  HEALTHCHECK    2  REBOOT    3  SHUTDOWN
```

## Reboot and shutdown

The two power operations differ in a small but deliberate way. `reboot`
(`src/server/handlers/reboot.rs:23`) records the request time, **builds the success reply, and then**
calls `mk_admin_reboot`:

```
  reboot(state, out, req):
      state.last_reboot_request_unix = now
      n = respond::status(out, req, 0)      // build the reply FIRST
      mk_admin_reboot()                      // then reset the machine
      return n
```

The reply is prepared before the reset because after `mk_admin_reboot` the machine restarts and the reply
would never be sent; the caller should not rely on receiving it regardless. `shutdown`
(`src/server/handlers/shutdown.rs:23`) instead calls `mk_admin_shutdown` and returns the syscall's result
as the status:

```
  shutdown(state, out, req):
      state.last_shutdown_request_unix = now
      rc = mk_admin_shutdown()
      return respond::status(out, req, rc)   // rc reported (if the machine is still up to send it)
```

Both are thin wrappers over the kernel's admin syscalls, which are the privileged operations that
actually reset or power off the machine; the [admin syscall family](../../subsystems/syscall/router.md)
is where the real work happens.

## State

`PowerState` (`src/state/mod.rs:17`) holds the last reboot and shutdown request timestamps. They are an
audit breadcrumb; nothing reads them back, and they accumulate without reset.

## Security analysis

The privileged operation, actually resetting the machine, is a kernel syscall gated by the capability the
power capsule holds. So the guarantee that only an authorized party can power the machine is that only
capsules granted the capability to reach the power service can request it, enforced by the kernel's
routing, not by a check inside this capsule.

The capability mask is `0x219` (`Capsule.mk`, `CAPSULE_REQUIRED_CAPS`), which decodes to CoreExec (1), IPC
(8), Memory (16), and Admin (512). Admin is the whole reason this capsule exists: `mk_admin_reboot` and
`mk_admin_shutdown` are privileged admin syscalls, and the kernel only honors them from a caller holding
the Admin capability, which is exactly the elevated bit in this mask. The least-privilege reading is that
Admin is the only power beyond the service baseline: it holds no Crypto, no FileSystem, no Network, and no
hardware capability at all, so the capsule that can reset the machine cannot read a key, write a file, open
a socket, or touch a device. That is the correct shape for a reset button, one privileged verb and nothing
else. This mask is the same `0x219` the [policy](policy.md) capsule holds, and for the same structural
reason: both hold exactly one Admin-class power and are otherwise minimal. The honest boundary, stated in
the [gaps](#honest-gaps), is that there is no caller attestation, so the real gate on who may power the
machine is the capability to *reach* port 4448, not any check inside the handler.

## Debugging

The service is `power` on port 4448 (`Capsule.mk`, `service:4448:power`), and the runner waits on that
fixed port with `mk_ipc_recv_from(4448, ...)` and replies directly to the attested sender
(`src/server/runner.rs:25`). Like [payment](payment.md), the power capsule is built into the image (the
Makefile includes `userland/capsule_power/Capsule.mk`) but is not spawned by the kernel init spawn plan, so
there is no `[POWER] capsule spawned` marker in the boot fleet; it is launched on demand and registers
under its manifest endpoint, so the test that it is up is whether `mk_service_lookup("power")` resolves
rather than a boot line. The behavior to expect when debugging a reboot that appears to hang is by design:
`reboot` builds its success reply *before* calling `mk_admin_reboot`, and the machine resets before that
reply can be delivered, so a caller should not wait on a reboot ack. `shutdown` is the opposite, it calls
`mk_admin_shutdown` and returns that syscall's `rc` as the status, so a nonzero status back from a shutdown
means the admin syscall itself refused (the caller reached the service but the kernel did not honor the
admin verb), which is the signature of a caller that reached port 4448 without the Admin authority the
syscall requires.

## Honest gaps

Stated plainly: the power capsule has **no caller attestation**, so any capsule that can reach port 4448
can request a reboot or shutdown; there is no debounce or rate limit, so repeated requests all fire; and
the recorded timestamps are never read. The real gate on who may power the machine is the capability to
reach this service, not logic here.

## Source map

```
  userland/capsule_power/src/server/runner.rs               the loop, reply-to-sender
  userland/capsule_power/src/server/handlers/reboot.rs       reply-then-reset
  userland/capsule_power/src/server/handlers/shutdown.rs     shutdown-then-reply
  userland/capsule_power/src/state/mod.rs                    the request timestamps
```

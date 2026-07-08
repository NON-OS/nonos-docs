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

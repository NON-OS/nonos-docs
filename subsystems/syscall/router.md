# The Syscall Router

Once a syscall has passed the [capability contract](boundary.md), the router dispatches it
to the handler for its family. This page documents the family dispatch, the handlers each
family routes to, and the fallback for an unrouted number. The code is under
`src/syscall/dispatch/router/`.

## The dispatch

The core is `dispatch_syscall` (`src/syscall/dispatch/router/dispatch_fn.rs:22`), a match
on the typed syscall number that routes each family to its handler:

```
  dispatch_syscall(syscall, a0..a5):
      Crypto* (the crypto calls)          -> crypto::dispatch_crypto
      admin::matches(nr)                  -> admin::handle
      microkernel_ops::matches(nr)        -> microkernel_ops::handle
      graphics_backend::matches(nr)       -> graphics_backend::handle
      MkSurface* and MkDisplayVsyncWait   -> surface_ops::handle
      MkInputEvent*                       -> input_ops::handle
      _                                   -> ENOSYS
```

The crypto, surface, and input families are matched by an explicit list of variants, while
the admin, microkernel, and graphics families are matched by a `matches` predicate each
module owns, so a family can claim its own numbers without the central match having to
enumerate every one. A number that no family claims falls through to `ENOSYS` (errno 38).
Because the [contract](boundary.md) already established that the caller holds the capability
this syscall requires, the router does not repeat the authority check; it only routes.

## The handlers

Each arm dispatches into a family module:

```
  crypto::dispatch_crypto   the in-kernel cryptographic primitives
  admin::handle             reboot, shutdown, policy push
  microkernel_ops::handle   the Mk* microkernel surface: ipc, memory, spawn,
                            time, capabilities, and the hardware broker
  graphics_backend::handle  display queries
  surface_ops::handle       surface register, share, attach, release, present, vsync
  input_ops::handle         input event post, drain, wait
```

The microkernel handler is the largest, since the `Mk*` family is the bulk of the syscall
surface; it does a second level of dispatch of its own to reach the individual IPC, memory,
process, device, and broker operations. The surface and input handlers connect to the
[graphics](../graphics/README.md) and [input](../input/README.md) subsystems, and the crypto handler to
the [crypto stack](../crypto/README.md).

## Counters and audit

The router is entered through `handle_syscall_dispatch`
(`src/syscall/dispatch/router/entry.rs`), which wraps `dispatch_syscall` with bookkeeping:
it counts total calls and their success, failure, and permission-denied outcomes, and
where a handler marks its result as requiring audit, it invokes the audit hook after the
call. The counters make the syscall surface observable, and the audit path records the
calls that ask to be recorded. The dispatch itself is the match above; this wrapper is the
accounting around it.

## Source

```
  src/syscall/dispatch/router/dispatch_fn.rs   dispatch_syscall, the family match
  src/syscall/dispatch/router/entry.rs          handle_syscall_dispatch, counters, audit
  src/syscall/dispatch/router/crypto.rs         the crypto family
  src/syscall/dispatch/router/admin/            the admin family
  src/syscall/dispatch/router/microkernel_ops.rs the Mk* family entry
  src/syscall/microkernel/                       the Mk* handler implementations
```

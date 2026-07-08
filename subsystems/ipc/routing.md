# Routing and Permission

An inbox is the destination; routing is how a message gets to one and who is allowed to
send it. A capsule names a service or a pid, not a queue, and the kernel resolves that name
to a destination inbox only after checking that the caller holds the capability the target
endpoint requires. This page documents the permission check, the destination resolution,
and the wake. The code is `src/ipc/kernel_ipc.rs` and the send syscalls under
`src/syscall/microkernel/ipc/`.

## The permission check

Every named-service route is gated by `kernel_check_ipc_permission`
(`src/ipc/kernel_ipc.rs:45`), which is also the first thing the routing path does:

```
  endpoint = lookup_service(target)          else ENOENT
  if not caps::has(caller_pid, endpoint.caps_required):
      return EACCES
```

A service is registered in the [service registry](../../security/capabilities-and-tokens.md)
with a `caps_required` mask, and a caller may only reach it if it holds those capabilities.
An unknown target is `ENOENT`; an unauthorized one is `EACCES`. The kernel does not deliver
first and check later, and it does not let a capsule reach a service by naming its inbox
directly, because the inbox name is derived from the endpoint by the kernel, not supplied by
the caller.

## Destination resolution

`kernel_route_ipc_corr` (`kernel_ipc.rs:57`) runs the same check and then decides the
destination inbox from the endpoint's owner:

```
  dest = if endpoint.pid == KERNEL_OWNER { target }      // kernel reply endpoint
         else { "proc.{endpoint.pid}" }                  // owning capsule inbox
  msg  = IpcMessage::new("proc.{caller_pid}", dest, data)
  msg.correlation = correlation
  try_enqueue_strict(dest, msg)
```

A capsule service routes into its owner's canonical `proc.<pid>` inbox; a kernel-owned
reply endpoint routes into the named endpoint inbox the kernel client drains. Either way,
the message envelope records `proc.<caller_pid>` as the sender, so the receiver learns who
sent it from the kernel's stamp rather than from a field the sender could forge; caller
attestation is the kernel's job, not the payload's. The optional correlation id is carried
through for request and reply matching.

## The wake

A successful enqueue to a capsule-owned endpoint checks whether the destination is asleep
and wakes it (`kernel_ipc.rs:83`):

```
  if endpoint.pid != KERNEL_OWNER and sched::is_sleeping(endpoint.pid):
      sched::wake_process(endpoint.pid)
```

This is what pairs with the receiver's [sleep loop](inbox.md): a receiver that found its
inbox empty went to sleep on a deadline, and the sender's wake makes the delivery prompt
rather than waiting for the receiver's next timed poll. The strict enqueue's three failure
modes map to errnos here, `ESRCH` for a missing or dead-owner destination and `EAGAIN` for a
full queue, so the sender learns the outcome.

## The send syscalls

Capsules reach this routing through the send family under `src/syscall/microkernel/ipc/`,
which adds the user-boundary handling on top:

- `sys_ipc_send` (`ipc/send.rs:47`) bounds the length against `MAX_MESSAGE_SIZE` before
  allocating, validates and copies the payload out of user space with the
  [usercopy](../memory/usercopy.md) checks, resolves the endpoint to a target name, verifies
  the caller satisfies the endpoint's capability, and routes. A reply to a service's own
  fixed reply endpoint is redirected to the original caller's private inbox via the pending
  reply table.
- `sys_ipc_send_to_pid` (`ipc/send_to_pid.rs:44`) delivers straight to a destination pid's
  `proc.<pid>` inbox for a server replying to a `recv_from` caller, still stamping the
  sender in the envelope, then wakes a sleeping destination.

Both reject a zero or oversize length up front, both validate the user buffer before
touching it, and both surface the strict-enqueue failure modes as errnos. The kernel copies
the payload into a kernel buffer before it enters the message, so the sender cannot mutate a
message the kernel has accepted.

## Source

```
  src/ipc/kernel_ipc.rs                    permission check, destination, wake
  src/syscall/microkernel/ipc/send.rs       sys_ipc_send, reply redirect
  src/syscall/microkernel/ipc/send_to_pid.rs sys_ipc_send_to_pid
  src/services/registry.rs                  lookup_service, endpoint caps_required
```

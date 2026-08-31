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

- `sys_ipc_send` (`ipc/send.rs:49`) bounds the length against `MAX_MESSAGE_SIZE` before
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

## The reply redirect and correlation

The subtle case is a server answering a request a client made with `mk_ipc_call`. A server
replies by sending to its own fixed reply endpoint, and `send_with_correlation` classifies
that send in `redirect_reply` (`send.rs:140`) into one of three destinations rather than
routing it as addressed:

```
  redirect_reply(sender_pid, target):                      # send.rs:140
      if target == sender's own reply_inbox:
          if pending_reply::pop(sender_pid) is Some(caller): Redirect::ToCaller
          else:                                             Redirect::ToReplyInbox
      else:                                                 Redirect::AsAddressed
```

`Redirect::ToCaller` (`send.rs:74`) is a reply owed to a capsule caller: the bytes go
straight into that caller's private reply inbox, the exact inbox its blocked `mk_ipc_call`
is draining, and the caller's pid is woken because a reply inbox has no owner the router
would wake on its own. `Redirect::ToReplyInbox` (`send.rs:107`) is a reply owed to a
kernel-mediated round trip (crypto pool, entropy, vfs, the block device): the kernel is the
one draining this inbox, so the bytes stay there. `Redirect::AsAddressed` (`send.rs:116`) is
every ordinary send, routed to its named target with its own correlation.

This three-way split is the fix for the reply-drop regression that killed crypto, nym, and
net: routing a reply back through the service resolver re-resolved the reply inbox to the
caller's own `proc.<pid>` inbox (the caller had adopted the endpoint), where the caller's
serve loop ate its own reply and the call timed out. Enqueuing directly into the private
reply inbox is what delivers the answer to the right place.

Correlation is what makes the redirect safe against forgery. `mk_ipc_call` mints a per-call
token that is monotonic and never zero (`call/sys_ipc_call.rs:36`), registers it in the
server's pending-reply table keyed by the caller's inbox (`sys_ipc_call.rs:72`), and stamps
it on the request. Both reply paths, the redirect above and the direct `mk_ipc_reply`
(`reply.rs:87`), read that same token back and stamp it on the reply. The blocked caller
runs `recv_reply_correlated` (`recv.rs:66`), which drains its inbox and delivers only the
message whose correlation equals the token it waits on, discarding everything else. A hostile
capsule can `mk_ipc_send` into another capsule's reply inbox, but `sys_ipc_send` hardcodes
correlation 0 (`send.rs:50`), which can never match a nonzero token, so a forged reply is
dropped rather than delivered. The server pairs a reply to its caller by the request's token,
recorded on every dequeue (`pending_reply::record_served`, `recv.rs:137`), not by queue
position, because kernel-side senders push no pending entry yet still occupy the inbox and
would otherwise shift the FIFO onto the wrong caller.

## Security analysis

Routing is where the reachability boundary is actually drawn: a capsule names a service or a pid, and
routing turns that name into a destination inbox only after deciding the caller is allowed to reach it.
Everything a capsule can do to another capsule passes through here. Three properties hold the boundary,
plus one honest limit.

**A name, never an address.** A caller reaches another capsule only by naming a registered endpoint;
there is no path where a capsule supplies a raw inbox name or an address. `resolve_send_target`
(`send.rs:151`) turns the numeric `endpoint` argument into a registry lookup, and
`kernel_route_ipc_corr` (`kernel_ipc.rs:69`) then derives the destination inbox itself: a capsule
service routes to `proc.<endpoint.pid>`, the owner's canonical inbox, and a kernel reply endpoint routes
to its own named inbox. The destination string is computed from the endpoint record, not taken from the
request, so a capsule cannot address an inbox the registry did not hand it, and cannot reach a service
by naming its inbox directly. The reachability graph is exactly the set of registered endpoints the
caller is permitted to call.

**Capability-gated before delivery, not after.** The first thing the route does is
`kernel_check_ipc_permission` / the inline check in `kernel_route_ipc_corr` (`kernel_ipc.rs:45`,
`kernel_ipc.rs:64`): an unknown target is `ENOENT`, and a caller that does not hold the endpoint's
`caps_required` is `EACCES`, before any enqueue. The send syscall re-checks the same requirement through
`caller_satisfies_endpoint` (`send_caps.rs:35`), which treats `caps_required == 0` as open to any sender
and otherwise requires the caller's permission bits to cover the mask. The kernel does not deliver first
and check later. Registering a name is separately gated from calling one: `sys_service_register`
(`register.rs:34`) demands the caller hold the register-service or admin right for an ordinary name
(`auth/caller_has_register_right.rs:17`) and refuses a reserved name outright (`register.rs:59`), so the
capability to advertise a service is not the capability to call it.

**The sender identity is the kernel's stamp.** The envelope's `from` is written as `proc.<caller_pid>`
by the route itself (`kernel_ipc.rs:74`), never supplied by the sender, so the receiver learns who
called it from the kernel's attestation rather than a forgeable field. A capsule cannot make a message
appear to come from another capsule.

**The payload is copied out of the sender before it enters the message.** Both send syscalls bound the
length against `MAX_MESSAGE_SIZE` up front (`send.rs:57`), validate the user buffer with the
[usercopy](../memory/usercopy.md) checks (`send.rs:60`), and copy it into a kernel buffer
(`send.rs:64`) before building the message. There is no unchecked user pointer crossing the boundary,
and once the kernel has accepted the message the sender cannot mutate it, because the kernel holds its
own copy.

The honest boundary: routing authenticates *who* the caller is and enforces *whether* it may reach an
endpoint, but it does not authenticate the *contents* of the payload beyond the envelope's integrity
tag, and it does not interpret them. A caller that legitimately holds the capability to reach a service
can send that service any bytes it likes within the size bound; whatever the receiving capsule does with
a well-formed but hostile request is that capsule's own input-validation problem, not routing's.

## Debugging routing

Every routing outcome is one of a small errno set, and the value tells you which stage refused the call.
A send that returns `EINVAL` (`-22`) was rejected before routing even ran, for a zero or oversize length
at the syscall's own bound (`send.rs:57`); `EFAULT` (`-14`) means the user buffer failed the usercopy
validate or copy (`send.rs:60`). Past those, the route's own returns are the interesting ones:
`ENOENT` (`-2`) means `lookup_service` found no endpoint for the target, so the name is wrong or the
service never registered; `EACCES` (`-13`) means the endpoint exists but the caller lacks its
`caps_required`, which is an authorisation problem, not a wiring one; `ESRCH` (`-3`) means the
destination inbox is missing or its owner is dead (the strict-enqueue `MissingInbox`/`DeadOwner` folded
together at `kernel_ipc.rs:90`); and `EAGAIN` (`-11`) means the destination queue is full. The
`sys_ipc_send` path also returns `EPERM` (`-1`) when `caller_satisfies_endpoint` refuses the caps
(`send.rs:76`), which is the send-syscall mirror of the route's `EACCES`, so both a `-1` and a `-13`
point at the same missing capability depending on which check caught it first.

Registration failures are a separate set: `sys_service_register` returns `EINVAL` for a bad name length
or non-UTF-8 name, `EPERM` for a reserved name or a caller without the register right, `EBUSY` (`-16`)
when the name or port is already taken (`RegError::Exists`), and `ENOMEM` (`-12`) when the table is at
`MAX_SERVICES = 256` (`register.rs:62`). This is how a rejected registration is told apart from a hung
call: a registration that never took shows up as one of these at the register syscall, while a hung call
is a send that returned `0` (routed fine) followed by a receiver that never gets the reply. For that
case the traces are the tool: the route prints `[ROUTE] from= target= dest= wake=` for the traced
destination pids (`kernel_ipc.rs:29`) with `wake=1` when it woke a sleeping receiver, and the send path
prints `[IPC-SEND] ...` (`send.rs:35`). A hung `mk_ipc_call` (`call/sys_ipc_call.rs:47`) that sent
successfully but timed out on the reply is diagnosed by whether the route to the *server* showed
`wake=1`: if the server was woken but no reply came back, the stall is in the server, and if the reply
send was refused, the caller's own reply inbox never filled and the call times out at its
5000 ms default (`call/sys_ipc_call.rs:87`).

## Source map

```
  src/ipc/kernel_ipc.rs                       permission check, destination resolution, wake, errno mapping
  src/syscall/microkernel/ipc/send.rs         sys_ipc_send, target resolution, reply redirect
  src/syscall/microkernel/ipc/send_to_pid.rs  sys_ipc_send_to_pid
  src/syscall/microkernel/ipc/send_caps.rs    caller_satisfies_endpoint, the send-side caps check
  src/syscall/microkernel/ipc/register.rs     sys_service_register and its errno set
  src/syscall/microkernel/ipc/call/sys_ipc_call.rs  the call/reply/timeout wrapper, per-call token
  src/syscall/microkernel/ipc/recv.rs         recv_reply_correlated, the correlation filter
  src/syscall/microkernel/ipc/reply.rs        sys_ipc_reply, the direct reply path
  src/syscall/microkernel/ipc/pending_reply/  the server pending-reply table, keyed by token
  src/services/registry.rs                    register_endpoint, lookup_service, lookup_port, MAX_SERVICES
  src/services/registry/auth/                 caller_can_register and the register-right check
```

Every reference above is verified against those trees. The capabilities these checks read are defined in
the [capability model](../../security/capabilities-and-tokens.md), the queues routing delivers into and
the wake it triggers are on the [inbox](inbox.md) page, the message it builds and stamps is on the
[envelope](envelope.md) page, and the user-buffer copy it performs is the [usercopy](../memory/usercopy.md)
boundary.

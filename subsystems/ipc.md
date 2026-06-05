# IPC

Capsules do not share memory except through brokered surfaces. Everything else
they need from each other and from kernel services travels as messages. The IPC
subsystem is named inboxes, a small message structure, and six syscalls. The
[architecture overview](../architecture/overview.md) covers it in section 10; the
[ABI reference](../abi/syscalls.md) lists the calls; here is how routing and
blocking actually work.

---

## Inboxes and messages

An inbox is a named multi-producer single-consumer ring
(`src/ipc/nonos_inbox.rs`). Names are stable strings:

```
  proc.<pid>        a process's own inbox, registered at spawn
  endpoint.<n>      a fixed numbered endpoint
  <service name>    a name registered in the service registry
```

A message is small and explicit (`src/ipc/nonos_channel.rs`):

```
  IpcMessage
    from         an envelope identifying the sender
    data         the payload bytes
    correlation  an id used to match a reply to its call
```

Routing between capsules goes through the kernel
(`src/ipc/kernel_ipc.rs`, `kernel_route_ipc_corr`), which resolves the target
name to an inbox and enqueues the message. The service registry
(`src/services/registry.rs`) maps service names to the endpoints that serve them,
so a client can find a server by name rather than by pid.

---

## The six primitives

```
  MkIpcSend       post to a named endpoint, do not wait
  MkIpcRecv       block on an endpoint until a message arrives or timeout
  MkIpcRecvFrom   like Recv, and also return the sender's pid
  MkIpcCall       synchronous request and reply over a private reply inbox
  MkIpcReply      reply to the caller pending on a private inbox
  MkIpcSendToPid  send straight to a pid's own inbox
```

### Send

`MkIpcSend` (`src/syscall/microkernel/ipc/send.rs:47`) validates the user buffer
is readable, copies it into a kernel-owned `Vec`, and routes it to the named
target. It returns without waiting. One subtlety: if the sender posts to its own
reply endpoint, the send is redirected to the private inbox of the caller
currently pending on it, which is how a server's reply reaches the right client.

### Receive

`MkIpcRecv` (`src/syscall/microkernel/ipc/recv.rs:44`) resolves the endpoint and
blocks until a message is available or the timeout expires. Blocking is real, not
a spin: with nothing waiting, the call computes a deadline and calls
`sleep_until`, yielding the CPU. A delivery into the inbox wakes it. It returns
the byte count copied, `ETIMEDOUT` if the deadline passes, or `ENOENT` if the
inbox does not exist.

`MkIpcRecvFrom` (`src/syscall/microkernel/ipc/recv_from.rs:58`) does the same and
additionally writes the sender's pid to a caller-supplied pointer, after
validating the pointer is writable. A server uses this to learn who called so it
can reply or send back directly with `MkIpcSendToPid`.

### Call and reply

`MkIpcCall` (`src/syscall/microkernel/ipc/call/sys_ipc_call.rs:27`) is the
request-and-reply primitive that clients and servers are built on:

```
  MkIpcCall(ep, req, req_len, resp, resp_len, timeout_ms)
    mint a private reply inbox for this caller
    push it onto the caller's pending-reply stack
    send the request to endpoint ep
    block on the private inbox (default 5 s if timeout_ms is 0)
    on reply or timeout, pop the pending-reply stack
    return the reply length, or an error
```

The private per-caller reply inbox is what keeps concurrent callers of the same
service from receiving each other's replies. Each call gets its own inbox; the
server's `MkIpcReply` routes the answer to exactly that inbox.

```
  client                         kernel routing                server
    |  MkIpcCall(ep, req) ----------->  mint private reply inbox
    |                                   route req to ep --------->  MkIpcRecvFrom
    |  (sleeping on reply inbox)                                    handle, build resp
    |                                <----------- MkIpcReply -------|
    |  <----- resp delivered to the private inbox, call returns
    v
```

### Direct send

`MkIpcSendToPid` (`src/syscall/microkernel/ipc/send_to_pid.rs`) posts straight to
a process's `proc.<pid>` inbox. A server that learned a caller's pid from
`MkIpcRecvFrom` uses this to push asynchronous messages back without the caller
having to be mid-call.

---

## Blocking is scheduler-backed

Every blocking IPC path uses the same mechanism as the rest of the kernel:
`sleep_until` to register a wake deadline and yield, and a delivery-side
`wake_process` to make the receiver runnable again
([scheduler](scheduler.md)). No IPC call busy-waits. A receiver with an empty
inbox consumes no CPU until either its timeout deadline arrives or a message is
routed to it. This is why a system of dozens of capsules, most of them parked in
`MkIpcRecv`, costs almost nothing when idle.

## Capability and naming

Every IPC call requires the `IPC` capability
([capabilities and tokens](../security/capabilities-and-tokens.md)). A capsule
that wants to be reachable by name registers with `MkServiceRegister`, which
requires the same capability and is checked at spawn against the endpoints the
capsule declared in its manifest, so a capsule cannot register a service it never
declared. Clients resolve names with `MkServiceLookup`.

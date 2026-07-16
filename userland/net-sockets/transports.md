# The transport backends

This page mirrors `src/clients/`. It is the outbound edge of the capsule: the shared IPC envelope every
backend call rides on, the TCP, UDP, and Nym clients, and how a handler picks a backend by socket kind.
`net.sockets` does no wire work of its own; everything a socket does on the network is a call from here to
`net.tcp`, `net.udp`, or `net.nym`. For which op calls which backend, read the [operations](operations.md)
page; for the port each backend is reached on, read the [state](state.md) page.

## The shared envelope

All three backends speak the same request-reply shape, and one function implements it: `envelope::call`
(`src/clients/envelope.rs:23`). It builds a twenty-byte header with the backend's magic, version 1, the op,
a zeroed errno and reserved word, a fixed `request_id` of 1, and the body length, appends the body, and
issues a synchronous `mk_ipc_call` to the backend port; then it parses the reply
(`src/clients/envelope.rs:24`, `src/clients/envelope.rs:35`). This is the same header layout the capsule's
own server uses, so a backend capsule and the sockets capsule share a wire format and differ only in the
magic word.

The parse is strict. A reply shorter than the header, or one whose magic does not match the backend's, is a
protocol error; a reply whose op does not echo the request, whose errno is nonzero, or whose declared
payload runs past the buffer or past the caller's output slice is rejected, returning the backend errno (or
a length error if the errno was zero) (`src/clients/envelope.rs:44`). On success it copies exactly the
declared payload bytes into the caller's output slice and returns the count
(`src/clients/envelope.rs:54`). A failed `mk_ipc_call` returns a fixed error word (`src/clients/envelope.rs:29`),
which is what a handler sees as `E_NO_TRANSPORT` after its own mapping. The upshot is that every backend
error, a refused connect, a full queue, an absent capsule, collapses at the handler into `E_NO_TRANSPORT`;
the sockets capsule does not forward the backend's own errno taxonomy to its client.

## The TCP client

`clients::tcp` is the `net.tcp` backend, keyed by the magic `NTCP` (`0x4E544350`)
(`src/clients/tcp.rs:21`). It exposes the connection-oriented op set the stream sockets need: `listen`,
`connect`, `accept`, `send`, `recv`, and `close`, with the op numbers matching the TCP capsule's protocol
(`src/clients/tcp.rs:22`). `connect` packs the four-byte destination IP and the `u16` port into a six-byte
body and returns the TCP connection handle (`src/clients/tcp.rs:33`); `listen` and `accept` likewise return
a `u32` handle through the `call_handle` helper, which insists the reply body is exactly four bytes
(`src/clients/tcp.rs:59`). `send` prepends the connection handle to the payload, and `recv` reads into the
caller's buffer and returns the byte count (`src/clients/tcp.rs:44`, `src/clients/tcp.rs:51`). These are the
calls behind `OP_LISTEN`, `OP_CONNECT`, `OP_ACCEPT`, and stream `OP_SEND` / `OP_RECV` / `OP_CLOSE`; the
handle they return becomes the socket's `transport_handle`.

## The UDP client

`clients::udp` is the `net.udp` backend, keyed by the magic `NUDP` (`0x4E554450`)
(`src/clients/udp.rs:21`). UDP is connectionless, so its op set is `bind`, `unbind`, `send`, and `recv`
addressed by local port rather than a connection handle (`src/clients/udp.rs:22`). `bind` and `unbind` each
carry only the `u16` local port (`src/clients/udp.rs:27`); `send` packs the local port, the four-byte
destination IP, the destination port, and then the payload into one body
(`src/clients/udp.rs:35`); `recv` carries the local port and reads into the caller's buffer
(`src/clients/udp.rs:44`). This is why a datagram socket never grows a `transport_handle`: `net.udp` is
keyed by the port pair the socket already holds in its local and remote addresses, so `OP_BIND` maps to
`udp::bind`, datagram `OP_SEND` to `udp::send`, `OP_RECV` to `udp::recv`, and `OP_CLOSE` to `udp::unbind`.

## The Nym client

`clients::nym` is the `net.nym` mixnet backend, and it is the one multi-file client because a mixnet
session is stateful in a way a datagram is not. Its magic is `NYM1` (`0x4E594D31`) and its op set is
`SET_GATEWAY`, `OPEN`, `SEND`, `RECV`, `COVER`, and `CLOSE` (`src/clients/nym/constants.rs:17`). The module
is split by concern (`src/clients/nym/mod.rs:17`):

- `config::set_gateway` packs the gateway IP, its port, and a one-byte enable flag and configures the
  requester before a session opens (`src/clients/nym/config.rs:20`).
- `session::open` opens a session and returns its `u32` id, and `session::close` tears it down by id
  (`src/clients/nym/session.rs:20`, `src/clients/nym/session.rs:26`).
- `transfer::send`, `transfer::recv`, and `transfer::cover` each carry the session id: `send` prepends it to
  the payload, `recv` reads into the caller's buffer, and `cover` requests a cover-traffic tick
  (`src/clients/nym/transfer.rs:22`, `src/clients/nym/transfer.rs:29`, `src/clients/nym/transfer.rs:33`).

`OP_CONNECT` on a mixnet socket runs `set_gateway` then `open`, storing the session id as the socket's
`transport_handle` (`src/server/handlers/connect.rs:59`); mixnet `OP_SEND` / `OP_RECV` / `OP_CLOSE` carry
that id to `transfer` and `session`; and the mixnet-only `OP_SETSOCKOPT` cover-tick maps to `transfer::cover`
(`src/server/handlers/setsockopt.rs:66`). The session id is the mixnet analogue of a TCP connection handle,
which is why both live in the same `transport_handle` field.

## Picking a backend

The choice is never global; it is per socket, made by branching on the `Socket.kind` inside each handler.
`send`, `recv`, and `close` each `match sock.kind` and route `Stream` to `tcp`, `Datagram` to `udp`, and
`Mixnet` to `nym` (`src/server/handlers/send.rs:41`, `src/server/handlers/recv.rs:42`,
`src/server/handlers/close.rs:34`); `connect` does the same to decide between a no-op for datagram, a Nym
open, and a TCP connect (`src/server/handlers/connect.rs:32`). The backend port each call needs comes from
the discovered service ports the [state](state.md) page describes, read fresh through `state::tcp()`,
`state::udp()`, and `state::nym()` at each call site. Because the kind is fixed at `OP_SOCKET` and the
handler reads it under the table lock, a socket always reaches the transport it was created for.

## Source map

```
  userland/capsule_net_sockets/src/clients/mod.rs        the client module declarations
  userland/capsule_net_sockets/src/clients/envelope.rs   the shared request/reply envelope over mk_ipc_call
  userland/capsule_net_sockets/src/clients/tcp.rs        the net.tcp client: listen, connect, accept, send, recv, close
  userland/capsule_net_sockets/src/clients/udp.rs        the net.udp client: bind, unbind, send, recv
  userland/capsule_net_sockets/src/clients/nym/mod.rs    the nym client module declarations
  userland/capsule_net_sockets/src/clients/nym/constants.rs  the NYM1 magic and op set
  userland/capsule_net_sockets/src/clients/nym/config.rs     set_gateway
  userland/capsule_net_sockets/src/clients/nym/session.rs    open and close
  userland/capsule_net_sockets/src/clients/nym/transfer.rs   send, recv, cover
  userland/capsule_net_sockets/src/server/handlers/send.rs, recv.rs, close.rs, connect.rs  the per-kind backend choice
  userland/capsule_net_sockets/src/server/handlers/setsockopt.rs  the mixnet cover-tick to nym
```

Every reference above is verified against those trees.

# Client networking libraries: sockets, TLS, HTTP

When a capsule wants to fetch something over the network it does not open a
socket in the kernel sense and it does not touch a device. It composes three
small libraries, each of which does one job and none of which trusts the others
to do theirs. From the bottom: `nonos_socket` carries bytes to and from the
network stack over IPC, `nonos_tls` turns that byte pipe into an authenticated
encrypted one, and `nonos_http` speaks HTTP/1.1 over whatever pipe it is handed.
The clean seam between them is that the two upper layers do no I/O of their own,
so they run identically over a real connection in the shell and over a buffer in
a test.

## nonos_socket: the transport

`userland/nonos_socket/` is the thin layer that talks to the `net.sockets`
capsule. A `TcpStream` here is not a kernel socket; it is a handle the sockets
capsule owns, driven entirely by request and reply over IPC. This crate touches
no device and no packet. Its whole job is to turn `connect`, `send`, `recv`, and
`close` into messages to `net.sockets` and answers back.

The surface (`src/lib.rs`) is exactly that: `lookup` to find the sockets
service, `open` / `connect_host` / `send` / `recv` / `close` as the operations
(`src/op/`), `TcpStream` as the handle that wraps them (`src/stream/`), and
`SocketError` for when the service says no. Because the transport is IPC, a
capsule that was not granted the Network capability cannot even reach
`net.sockets`, so the authority check happens below this crate, not inside it.

## nonos_tls: the secure pipe

`userland/nonos_tls/` sits on top of a `TcpStream` and turns it into a TLS 1.3
session: one round-trip handshake, certificate chain verification to pinned
roots, and AEAD-sealed records. It is documented in full in
[../../subsystems/networking/tls.md](../../subsystems/networking/tls.md); the
thing to know here is that it consumes the socket's byte pipe and exposes another
byte pipe, so the layer above it does not know or care that encryption happened.

## nonos_http: the protocol

`userland/nonos_http/` is HTTP/1.1 for a client, and its defining property is
that it has no I/O of its own. Requests are built into bytes and responses are
parsed from bytes. From `src/lib.rs`: `Request` and `RequestBuilder`
(`src/request/`) assemble a request and serialise it, `parse_response` and
`Response` (`src/response/`) read a reply back, including chunked transfer
decoding (`src/response/chunk/`), and `fetch` with `Stream` (`src/stream.rs`) is
the convenience that drives a whole request and response over a stream you hand
it. `HttpError` is the failure surface.

Because parsing is pure, the crate is tested against fixed byte inputs, including
malformed ones (`tests/response.rs`, `tests/request.rs`, `tests/damage.rs`). A
response parser that only ever sees well-formed input is a parser you do not yet
trust; the damage test is there to make sure a hostile or truncated reply fails
cleanly instead of misparsing.

## How they compose

A plain HTTP fetch is `nonos_socket` plus `nonos_http`: connect a `TcpStream`,
hand it to `fetch`, get a `Response`. An HTTPS fetch inserts `nonos_tls` in the
middle: connect the `TcpStream`, wrap it in a TLS session, then run the same
`fetch` over the TLS pipe. The HTTP code is identical in both cases, which is the
point of keeping I/O out of it. This is the composition the browser and the
wallet use, and it is why the certificate check is the same code for both (see
the TLS page for why that mattered).

## Status

| Library | Source | Status |
|---|---|---|
| `nonos_socket` (IPC to net.sockets) | `userland/nonos_socket/src/` | IMPLEMENTED; DEMONSTRATED under QEMU |
| `nonos_http` (HTTP/1.1 client, pure bytes) | `userland/nonos_http/src/` | IMPLEMENTED; TESTED (incl. malformed input) |
| `nonos_tls` (TLS 1.3) | `userland/nonos_tls/src/` | see the TLS page |

## Source

`userland/nonos_socket/src/lib.rs` for the transport surface,
`userland/nonos_http/src/lib.rs` for the protocol, and the request and response
directories for the assembly and parsing. The TLS layer is `userland/nonos_tls/`.

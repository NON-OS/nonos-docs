# The wire protocols

This page mirrors `src/protocol/` and `src/register.rs`. It documents the four binary protocols clients
speak to `net_core`, the shared header they all use, the op set and errno set per protocol, and the service
names and ports the capsule registers so a client can find each protocol. For the loop that receives these
requests and the handlers that answer them, read the [server](server.md) page; for the identity and the
capability mask, read the [README](README.md).

## One header, four magics

Every request and reply is a 20-byte little-endian header optionally followed by a payload. The header
layout is fixed (`src/protocol/header.rs:19`): a 4-byte magic, a 2-byte version (always 1), a 2-byte op, a
2-byte status or reserved field, a 2-byte reserved field, a 4-byte request id, and a 4-byte payload length.
`HDR_LEN` is 20 (`src/protocol/ops.rs:20`, `src/server/parse_req.rs:19`).

The magic selects which protocol the request belongs to, and the server dispatches on it
(`src/server/runner/mod.rs`):

| Magic | Value | Protocol | Source |
|---|---|---|---|
| `NNET` | `0x4E4E4554` | device link (driver-facing, not a registered service) | `src/protocol/ops.rs:17` |
| `NTCP` | `0x4E544350` | TCP sockets | `src/protocol/tcp.rs:17` |
| `NUDP` | `0x4E554450` | UDP sockets | `src/protocol/udp.rs:17` |
| `NDNS` | `0x4E444E53` | DNS resolver | `src/protocol/dns.rs:17` |
| `NDHC` | `0x4E444843` | DHCP lease status | `src/protocol/ops.rs:18` |

The request writer and the response reader for the driver-facing `NNET` protocol are
`write_request` and `parse_response` (`src/protocol/header.rs:19`, `src/protocol/header.rs:33`); the
server side parses client requests with `parse` and encodes replies with `reply`, both documented on the
[server](server.md) page. A reply reuses the same header with the client's magic and op echoed back and the
status field set to an errno (`src/server/respond.rs:21`).

## The services and their ports

After setup succeeds, `register::all` registers five service names, each at its own port, and prints a
marker (`src/register.rs:41`). These are the ports a client looks up to reach each protocol.

| Service | Port | Magic to use | Source |
|---|---|---|---|
| `net.tcp` | 4476 | `NTCP` | `src/register.rs:19`, `src/register.rs:25` |
| `net.udp` | 4472 | `NUDP` | `src/register.rs:20`, `src/register.rs:26` |
| `net.dhcp.client` | 4474 | `NDHC` | `src/register.rs:21`, `src/register.rs:27` |
| `net.dns` | 4478 | `NDNS` | `src/register.rs:22`, `src/register.rs:28` |
| `net.ip` | 4479 | `NIP4` | `src/register.rs:23`, `src/register.rs:29` |

If all five register the capsule logs `[NET-CORE] registered net.tcp net.udp net.dhcp.client net.dns net.ip`;
otherwise it logs `[NET-CORE] registration partial failure` (`src/register.rs:52`). All five services are
served from the one service inbox in the request loop, so a client sends to the registered port and the
capsule routes by magic, `NDHC`, `NTCP`, `NUDP`, `NDNS`, or `NIP4` (`src/server/runner/dispatch.rs:30`).

## TCP (`NTCP`, service `net.tcp`)

The op constants are in `src/protocol/tcp.rs:19`.

| Op | Value | Request payload | Reply payload | Source |
|---|---|---|---|---|
| `OP_CONNECT` | 3 | 4-byte IPv4 + 2-byte port | 4-byte app handle | `src/protocol/tcp.rs:20`, `src/server/handlers/tcp/connect.rs:43` |
| `OP_SEND` | 5 | 4-byte handle + data | 4-byte bytes-accepted | `src/protocol/tcp.rs:22`, `src/server/handlers/tcp/send.rs:25` |
| `OP_RECV` | 6 | 4-byte handle | received bytes | `src/protocol/tcp.rs:23`, `src/server/handlers/tcp/recv.rs:25` |
| `OP_CLOSE` | 7 | 4-byte handle | none | `src/protocol/tcp.rs:24`, `src/server/handlers/tcp/close.rs:25` |
| `OP_STATE` | 9 | 4-byte handle | 1-byte state code | `src/protocol/tcp.rs:25`, `src/server/handlers/tcp/state.rs:25` |

`OP_LISTEN` (2) and `OP_ACCEPT` (4) are defined in the protocol module (`src/protocol/tcp.rs:19`,
`src/protocol/tcp.rs:21`) but are not wired into the TCP dispatch, which handles only connect, send, recv,
close, and state (`src/server/handlers/tcp/mod.rs:29`). The 1-byte state code returned by `OP_STATE` maps
the smoltcp `tcp::State` enum: 0 Listen, 1 SynSent, 2 SynReceived, 3 Established, 4 CloseWait, 5 FinWait1,
6 FinWait2, 7 Closing, 8 TimeWait, 9 LastAck, and `0xFF` Closed (`src/server/handlers/tcp/state.rs:49`).

TCP errnos (`src/protocol/tcp.rs:27`): `E_OK` 0, `E_BAD_OP` 3, `E_BAD_LEN` 4, `E_NO_SOCKET` 5,
`E_RX_EMPTY` 11, `E_NOT_CONNECTED` 12.

## UDP (`NUDP`, service `net.udp`)

The op constants are in `src/protocol/udp.rs:19`.

| Op | Value | Request payload | Reply payload | Source |
|---|---|---|---|---|
| `OP_BIND` | 2 | 2-byte local port | none | `src/protocol/udp.rs:19`, `src/server/handlers/udp/bind.rs:32` |
| `OP_UNBIND` | 3 | 2-byte local port | none | `src/protocol/udp.rs:20`, `src/server/handlers/udp/unbind.rs:23` |
| `OP_SEND` | 4 | 2-byte local port + 4-byte IPv4 + 2-byte port + data | none | `src/protocol/udp.rs:21`, `src/server/handlers/udp/send.rs:26` |
| `OP_RECV` | 5 | 2-byte local port | 4-byte src IPv4 + 2-byte src port + data | `src/protocol/udp.rs:22`, `src/server/handlers/udp/recv.rs:26` |

UDP errnos (`src/protocol/udp.rs:24`): `E_OK` 0, `E_BAD_OP` 3, `E_BAD_LEN` 4, `E_NO_SOCKET` 5,
`E_BIND_FAILED` 6, `E_RX_EMPTY` 8, `E_NOT_CONNECTED` 12.

## DNS (`NDNS`, service `net.dns`)

One op, `OP_RESOLVE_A` (2) (`src/protocol/dns.rs:19`). The request payload is the hostname as raw UTF-8
bytes; the reply payload on success is the 4-byte IPv4 address of the first A record
(`src/server/handlers/dns/resolve_a.rs:29`, `src/server/handlers/dns/resolve_a.rs:59`). DNS errnos
(`src/protocol/dns.rs:21`): `E_OK` 0, `E_BAD_OP` 3, `E_NAME_INVALID` 9, `E_SERVFAIL` 10, `E_NO_LEASE` 11.
`E_NO_LEASE` is returned when there is no DHCP lease yet, so no DNS socket has been installed
(`src/server/handlers/dns/resolve_a.rs:39`); `E_SERVFAIL` covers a query timeout or an empty answer
(`src/server/handlers/dns/resolve_a.rs:50`).

## DHCP status (`NDHC`, service `net.dhcp.client`)

One op, `OP_LEASE_STATUS` (3) (`src/protocol/ops.rs:27`). It takes no request payload and returns an
18-byte body (`src/server/handlers/dhcp_status.rs:32`): a 1-byte state (3 bound, 1 unbound), then when
bound the 4-byte IPv4 address, a 1-byte prefix length, the 4-byte gateway, the 4-byte DNS server, and a
4-byte lease-seconds field. When unbound only the leading state byte is set (`src/server/handlers/dhcp_status.rs:43`).
This op shares the errno set of the base protocol (`src/protocol/errno.rs:17`): `E_OK` 0, `E_BAD_MAGIC` 1,
`E_BAD_VERSION` 2, `E_BAD_OP` 3, `E_BAD_LEN` 4.

## Health (op 1, any magic)

`OP_HEALTHCHECK` (1) is checked before the magic dispatch, so it answers regardless of protocol
(`src/server/runner/mod.rs`, `src/server/handlers/health.rs:21`). It takes no payload and replies `E_OK`
with the request's own magic and op echoed back (`src/server/handlers/health.rs:23`).

## The driver-facing NNET protocol

`NNET` is not a service clients call; it is the protocol `net_core` speaks as a client to the NIC driver.
Its ops are `OP_LINK_STATUS` 2, `OP_MAC_ADDRESS` 3, `OP_TX_PACKET` 4, and `OP_RX_PACKET` 5
(`src/protocol/ops.rs:22`). The [device](device.md) page documents how each is used.

## Source map

```
  userland/capsule_net_core/src/protocol/header.rs   write_request, parse_response, the 20-byte layout
  userland/capsule_net_core/src/protocol/ops.rs      HDR_LEN, MAGIC_NNET/NDHC, the NNET and lease ops
  userland/capsule_net_core/src/protocol/tcp.rs      MAGIC_NTCP, the TCP ops and errnos
  userland/capsule_net_core/src/protocol/udp.rs      MAGIC_NUDP, the UDP ops and errnos
  userland/capsule_net_core/src/protocol/dns.rs      MAGIC_NDNS, OP_RESOLVE_A, the DNS errnos
  userland/capsule_net_core/src/protocol/errno.rs    the base errno set behind NDHC and header faults
  userland/capsule_net_core/src/register.rs          the four net.* service names and their ports
  userland/capsule_net_core/src/server/runner.rs     the magic dispatch and the health short-circuit
  userland/capsule_net_core/src/server/respond.rs    the reply encoder that echoes magic and op
  userland/capsule_net_core/src/server/handlers/     the per-op handlers cited per row above
```

Every reference above is verified against those trees.

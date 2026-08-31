# The Layered Stack

The network stack ships in two forms. The consolidated form is one capsule, `net_core`, running
smoltcp over a NIC bridge (the [stack](stack.md) page). The decomposed form is one capsule per
protocol layer, each hand-written no_std code, talking to its neighbours over kernel-mediated IPC:
L2/ARP, IPv4/ICMP, UDP, TCP, the sockets facade, DNS, and DHCP. This page documents the decomposed
form against the source, with `file:line` references.

The two forms are mutually exclusive. The per-layer capsules compile out when the
`nonos-capsule-net-core` feature is set (`src/userspace/init/spawn_plan/network/mod.rs:19`), and
`net_core` registers the same service names (`net.tcp`, `net.udp`, `net.dns`, `net.ip`,
`net.dhcp.client`) that the split capsules provide, so exactly one of the two runs at a time. Clients
resolve services by name and get whichever stack is up; the sockets facade, nym, and socks5 are
agnostic and run above either.

## The common shape

Every layer capsule has the same skeleton: `_start` initialises the heap, `setup` resolves the layer
directly beneath it through the kernel service registry (`mk_service_lookup(name, &port, &pid)`), and
`server::run()` loops on `mk_ipc_recv_from`, parses a 20-byte header, and dispatches on the op. The
split capsules do not self-register; their endpoint names are registered by the kernel at spawn under
the network spawn plan. Downstream calls are synchronous `mk_ipc_call`. Op numbers are per-capsule
discriminants scoped by each capsule's own wire magic, so op 3 is `SET_CONFIG` on `net.ip` but
`LEASE_STATUS` on `net.dhcp.client`; there is no global op namespace.

The data path is a chain of IPC clients, each layer holding a `*_client` module for the one below:

```
  driver (virtio_net/e1000/rtl)
     ^  nic_client (NNET frames)
  net.l2   ARP + Ethernet
     ^  l2_client (ARP_RESOLVE, SEND_FRAME)
  net.ip   IPv4 + ICMP + routes
     ^  ip_client (SEND_PACKET / POLL_PACKET, proto 6 or 17)
  net.udp / net.tcp
     ^  clients/{tcp,udp,dns,nym}
  net.sockets   BSD-like facade
     ^
  applications (nonos_std TcpStream/UdpSocket)

  net.dhcp.client  composes raw Eth/IP/UDP/BOOTP, drives net.l2 + net.ip
  net.dns          net.udp for queries, net.dhcp.client for the upstream address
```

## L2: Ethernet and ARP (`capsule_net_l2`)

Ops (`src/protocol/ops.rs:21`, dispatch `src/server/runner.rs:44`): `HEALTHCHECK 1`, `GET_MAC 2`,
`GET_LINK 3`, `SEND_FRAME 4`, `POLL_FRAME 5`, `ARP_RESOLVE 6`, `SET_IP 7`.

Below it is the NIC driver, reached through `nic_client/` on a separate `NNET` envelope
(`nic_client/wire.rs:24`: `OP_MAC_ADDRESS 3`, `OP_TX_PACKET 4`, `OP_RX_PACKET 5`). Setup probes NIC
candidates in order `driver.virtio_net0`, `driver.e1000_0`, `driver.rtl8169_0`, `driver.rtl8139_0`,
first hit wins (`setup/discover.rs:22`).

`ARP_RESOLVE` (`server/handlers/arp_resolve.rs:28`) returns the 6-byte MAC on a cache hit; on a miss
it records the target as pending, broadcasts an ARP request, and replies `E_NO_NEIGHBOUR` so the
caller retries. Inbound ARP is learned under a policy (`arp/cache/learn.rs:24`): refresh a matching
entry, reject a rebind to a different MAC, insert only if the reply was solicited. The cache holds 64
entries and 8 pending requests (`arp/cache/constants.rs:17`); it has **no time-based TTL**, evicting
the least-recently-updated entry by monotonic sequence only when full (`arp/cache/evict_oldest.rs:20`).

## IP: IPv4 and ICMP (`capsule_net_ip`)

Ops (`src/protocol/ops.rs:21`): `HEALTHCHECK 1`, `GET_CONFIG 2`, `SET_CONFIG 3`, `SEND_PACKET 4`,
`POLL_PACKET 5`, `ROUTE_ADD 6`, `ROUTE_CLEAR 7`. Below it is `net.l2` via `l2_client/`
(`ARP_RESOLVE`, `SEND_FRAME`).

Checksums are RFC 1071 one's-complement (`ipv4/checksum.rs:22`). Egress does a route lookup, ARP-
resolves the gateway-or-destination MAC, and ships the frame through `l2_client`
(`egress/send.rs:26`). The routing table is 16 entries with longest-prefix match and a default-route
fallback (`route/table.rs`). **There is no fragmentation**: outbound always sets the Don't-Fragment
bit (`ipv4/build.rs:56`) and inbound rejects any fragment (`ipv4/parse.rs:58`), so there is no
reassembly. ICMP echo is answered internally: an inbound echo request is replied to before it reaches
a client and reported as absorbed (`icmp/responder.rs:37`).

## UDP (`capsule_net_udp`)

Ops (`src/protocol/ops.rs:17`): `HEALTHCHECK 1`, `BIND 2`, `UNBIND 3`, `SEND 4`, `RECV 5`. Below it is
`net.ip` via `ip_client/`, sending with protocol 17 (`ip_client/wire.rs:29`). The UDP checksum covers
the IPv4 pseudo-header per RFC 768, mapping a zero result to 0xFFFF (`udp/checksum.rs:39`). The bind
table holds 64 ports, one owner per port keyed by pid (`state/table.rs:21`, `:41`).

## TCP (`capsule_net_tcp`)

Ops (`src/protocol/ops.rs:17`): `HEALTHCHECK 1`, `LISTEN 2`, `CONNECT 3`, `ACCEPT 4`, `SEND 5`,
`RECV 6`, `CLOSE 7`, `SHUTDOWN 8`, `STATE 9`. Below it is `net.ip` with protocol 6. The run loop drives
timers each iteration (`server/runner.rs:37`).

This is a real TCP, not a shim. The state machine covers Listen, SynSent, SynReceived, Established,
CloseWait, FinWait1/2, Closing, TimeWait, and LastAck (`tcp/state.rs:19`), with the transitions in
`server/tcp_rx/transitions/` including the client three-way handshake (`handshake.rs:23`), TimeWait
armed for 2*MSL (`transitions/closing.rs:49`), and RST handling (`tcp_rx/rst.rs`). It implements:

- **Reno congestion control** (`tcp/cc.rs:19`): slow-start, congestion avoidance, `on_rto` halving,
  and fast retransmit/recovery at three duplicate ACKs.
- **RTT estimation** (Jacobson/Karels, RFC 6298, `tcp/rtt.rs:35`) with an RTO clamp and exponential
  backoff.
- **Retransmission** on the RTO timer, aborting after `MAX_RETX` (`server/retransmit.rs:22`).
- **Out-of-order reassembly** in a bounded `BTreeMap` keyed by sequence (`state/reasm/`).
- **ISS generation, RFC 6528 style** (`tcp/iss.rs:20`): `now_ms + SipHash-2-4(key, four-tuple)`, the
  SipHash key seeded from 16 CSPRNG bytes at boot (`setup.rs:31`, `tcp/siphash/`).

## Sockets facade (`capsule_net_sockets`)

Ops (`src/protocol/ops.rs:17`): `HEALTHCHECK 1`, `SOCKET 2`, `BIND 3`, `LISTEN 4`, `ACCEPT 5`,
`CONNECT 6`, `SEND 7`, `RECV 8`, `CLOSE 9`, `GETSOCKOPT 10`, `SETSOCKOPT 11`, `CONNECT_HOST 12`,
`POLL 13`, `CONNECT_NB 14`. It resolves `net.tcp`, `net.udp`, `net.dns`, and lazily `net.nym`
(`state.rs:21`).

`SOCKET` sets the socket kind from the request: family 4 (AF_INET) and kind `1 = Stream`,
`2 = Datagram`, `3 = Mixnet` (`server/handlers/socket.rs:24`). `connect` dispatches on that kind
(`connect/handle.rs:35`): Stream to `net.tcp` (`update_stream.rs:33`), Datagram to `net.udp`, Mixnet
to `net.nym` with no IP transport (`update_mixnet.rs:33`). `CONNECT_HOST` first resolves the name,
either a literal IPv4 or `net.dns` `RESOLVE_A` (`connect/resolve_host.rs:21`). The mixnet socket path
gives a byte-stream illusion over datagrams: each write is fragmented into fixed-size mix frames
(`mixnet_send.rs:32`, `mixnet_frame/encode.rs`), and leftover bytes from a decoded frame are held in a
residual buffer (`mixnet_residual/`). This is the surface `nonos_std`'s `TcpStream` and `UdpSocket`
bind to.

## DNS (`capsule_net_dns`)

Ops (`src/protocol/ops.rs:17`): `HEALTHCHECK 1`, `RESOLVE_A 2`, `RESOLVE_AAAA 3`, `FLUSH_CACHE 4`,
`SET_UPSTREAM 5`. Below it is `net.udp` via `udp_client/`. `resolve_a` checks the cache, and on a miss
builds a query, exchanges it with a 3 s deadline and 400 ms resend matching on transaction id and
question (`server/handlers/resolve_common.rs:36`), and caches the answer with the **TTL from the
response record** (`resolve_a.rs:44`). The cache holds 128 entries with 64-byte names and honours the
TTL on lookup (`dns/cache/entry.rs:19`, `ops.rs:23`). The **upstream resolver comes from the DHCP
lease**: setup calls `net.dhcp.client` `LEASE_STATUS` and installs its DNS field
(`dhcp_upstream.rs:22`), overridable at runtime via `SET_UPSTREAM`.

## DHCP (`capsule_net_dhcp`)

Ops (`src/protocol/ops.rs:17`): `HEALTHCHECK 1`, `LEASE_REQUEST 2`, `LEASE_STATUS 3`,
`LEASE_RELEASE 4`, `LEASE_RENEW 5`. It drives **both** `net.l2` and `net.ip` (`setup/discover.rs:34`).
Because there is no IP address during acquisition, DHCP composes raw Ethernet/IPv4/UDP/BOOTP broadcast
frames itself (`frame/`) and sends them through `net.l2` `SEND_FRAME`.

The DORA flow (`dora/acquire.rs:29`): DISCOVER in the Selecting state waiting for an OFFER
(`dora/discover.rs:31`), REQUEST in the Requesting state waiting for an ACK (a NAK is an error,
`dora/request.rs`), then install (`dora/install.rs:31`): the subnet mask is converted to a prefix and
the lease pushed to `net.ip` `SET_CONFIG` (IP + prefix + gateway, `ip_client/set_config.rs:34`) and to
`net.l2` `SET_IP` so L2 can answer ARP for the host (`l2_client/set_ip.rs:36`); the state goes Bound.
The lease's DNS server stays in DHCP capsule state and is read by the DNS capsule via `LEASE_STATUS`.
Release sends DHCPRELEASE and clears the `net.ip` config; renew re-runs the request.

## Proof and status

The decomposed capsules' untrusted-input parsers are host-tested in `userland/net_proofs/` (16 test
functions): the DNS response parser terminates on every two-byte compression pointer including the
self-referential loop, ICMP and ARP parsing never panic and reject truncation, the TCP segment parser
and out-of-order reassembly never panic on hostile streams, and the DHCP option walk never panics
(`userland/net_proofs/README.md`). These are safety properties (no panic, no out-of-bounds,
termination) on the parsers, not end-to-end functional proofs.

| Layer | Status |
|---|---|
| L2 Ethernet + ARP, cache eviction | IMPLEMENTED; ARP parser TESTED (net_proofs) |
| IPv4 checksum, routing, ICMP echo | IMPLEMENTED; ICMP/IPv4 parsers TESTED |
| IPv4 fragmentation / reassembly | NOT IMPLEMENTED (DF set, fragments rejected) |
| UDP | IMPLEMENTED |
| TCP state machine, Reno CC, RTT/RTO, reassembly, RFC 6528 ISS | IMPLEMENTED; segment + reassembly parsers TESTED |
| Sockets facade incl. mixnet stream illusion | IMPLEMENTED |
| DNS resolver + cache, DHCP-sourced upstream | IMPLEMENTED; response parser TESTED |
| DHCP DORA, lease install to net.ip + net.l2 | IMPLEMENTED; reply parser TESTED |
| Consolidated `net_core` to a bound lease | PROVEN under QEMU (`DHCP BOUND 10.0.2.15`, see [services](services.md)) |

## Source

```
  userland/capsule_net_l2/src/       Ethernet, ARP cache, nic_client, ops
  userland/capsule_net_ip/src/       IPv4, ICMP responder, route table, l2_client
  userland/capsule_net_udp/src/      UDP checksum, bind table, ip_client
  userland/capsule_net_tcp/src/      TCP state machine, cc, rtt, retransmit, reasm, siphash ISS
  userland/capsule_net_sockets/src/  socket table, connect dispatch, clients/, mixnet frames
  userland/capsule_net_dns/src/      DNS parse, cache, udp_client, dhcp_upstream
  userland/capsule_net_dhcp/src/     DORA, raw frame compose, l2_client + ip_client
  userland/net_proofs/               host-runnable adversarial parser proofs
  src/userspace/init/spawn_plan/network/mod.rs   the consolidated-vs-decomposed feature gate
```

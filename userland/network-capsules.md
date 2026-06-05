# Network Capsules

This page documents the userland network capsule stack. Read
[Capsule Inventory](capsules.md) and [Networking](../subsystems/networking.md)
first.

---

## 1. Boot order

The network spawn plan starts L2, IP, UDP, DHCP, TCP, DNS, Nym, and sockets in
that order (`src/userspace/init/spawn_plan/network.rs:17`). Each network
capsule is a no_std service with a `main.rs` entrypoint and a protocol ops
table in `src/protocol/ops.rs`.

```
  +-----------+
  | net.l2    |
  +-----+-----+
        |
  +-----+-----+
  | net.ip    |
  +-----+-----+
        |
  +-----+-----+
  | udp tcp   |
  +-----+-----+
        |
  +-----+-----+
  | dhcp dns  |
  +-----+-----+
        |
  +-----+-----+
  | sockets   |
  | nym       |
  +-----------+
```

## 2. Capsule contracts

| Capsule | Service | Caps | Protocol operations | Entrypoint | Spec refs |
|---------|---------|------|---------------------|------------|-----------|
| `net.l2` | `service:4400:net.l2` | `0x00019` | healthcheck, get MAC, get link, send frame, poll frame, ARP resolve | `userland/capsule_net_l2/src/main.rs:34` | `userland/capsule_net_l2/Capsule.mk:13`, `userland/capsule_net_l2/Capsule.mk:16`, `userland/capsule_net_l2/src/protocol/ops.rs:21` |
| `net.ip` | `service:4402:net.ip` | `0x00019` | healthcheck, get config, set config, send packet, poll packet, route add, route clear | `userland/capsule_net_ip/src/main.rs:36` | `userland/capsule_net_ip/Capsule.mk:14`, `userland/capsule_net_ip/Capsule.mk:17`, `userland/capsule_net_ip/src/protocol/ops.rs:21` |
| `net.udp` | `service:4420:net.udp` | `0x00019` | healthcheck, bind, unbind, send, receive | `userland/capsule_net_udp/src/main.rs:32` | `userland/capsule_net_udp/Capsule.mk:12`, `userland/capsule_net_udp/Capsule.mk:14`, `userland/capsule_net_udp/src/protocol/ops.rs:17` |
| `net.dhcp.client` | `service:4440:net.dhcp.client` | `0x00019` | healthcheck, lease request, lease status, lease release, lease renew | `userland/capsule_net_dhcp/src/main.rs:35` | `userland/capsule_net_dhcp/Capsule.mk:12`, `userland/capsule_net_dhcp/Capsule.mk:14`, `userland/capsule_net_dhcp/src/protocol/ops.rs:17` |
| `net.tcp` | `service:4430:net.tcp` | `0x00019` | healthcheck, listen, connect, accept, send, receive, close, shutdown | `userland/capsule_net_tcp/src/main.rs:32` | `userland/capsule_net_tcp/Capsule.mk:12`, `userland/capsule_net_tcp/Capsule.mk:14`, `userland/capsule_net_tcp/src/protocol/ops.rs:17` |
| `net.dns` | `service:4450:net.dns` | `0x00019` | healthcheck, resolve A, resolve AAAA, flush cache, set upstream | `userland/capsule_net_dns/src/main.rs:32` | `userland/capsule_net_dns/Capsule.mk:11`, `userland/capsule_net_dns/Capsule.mk:13`, `userland/capsule_net_dns/src/protocol/ops.rs:17` |
| `net.sockets` | `service:4460:net.sockets` | `0x00019` | healthcheck, socket, bind, listen, accept, connect, send, receive, close, getsockopt, setsockopt | `userland/capsule_net_sockets/src/main.rs:31` | `userland/capsule_net_sockets/Capsule.mk:12`, `userland/capsule_net_sockets/Capsule.mk:14`, `userland/capsule_net_sockets/src/protocol/ops.rs:17` |
| `net.nym` | `service:4470:net.nym` | `0x00039` | healthcheck, set gateway, open session, send, receive, cover tick, close, set topology, set credential, create SURB, send reply, set timing, set authority, sync directory, topology status, timing status | `userland/capsule_net_nym/src/main.rs:37` | `userland/capsule_net_nym/Capsule.mk:11`, `userland/capsule_net_nym/Capsule.mk:13`, `userland/capsule_net_nym/src/protocol/ops.rs:17` |

## 3. Server loops

The network capsules wait for setup before entering their server loops. L2,
IP, UDP, TCP, DNS, sockets, Nym, and DHCP all call `wait_for_setup()` before
`server::run()` (`userland/capsule_net_l2/src/main.rs:34`,
`userland/capsule_net_ip/src/main.rs:36`,
`userland/capsule_net_udp/src/main.rs:32`,
`userland/capsule_net_tcp/src/main.rs:32`,
`userland/capsule_net_dns/src/main.rs:32`,
`userland/capsule_net_sockets/src/main.rs:31`,
`userland/capsule_net_nym/src/main.rs:37`,
`userland/capsule_net_dhcp/src/main.rs:35`). The handler surface matches the
op tables: L2 handlers cover link, MAC, frame TX/RX, and ARP
(`userland/capsule_net_l2/src/server/handlers/mod.rs:17` to
`userland/capsule_net_l2/src/server/handlers/mod.rs:22`); IP handlers cover
configuration, send, poll, and routes
(`userland/capsule_net_ip/src/server/handlers/mod.rs:17` to
`userland/capsule_net_ip/src/server/handlers/mod.rs:23`); UDP, TCP, and
sockets handlers cover their socket-family operations
(`userland/capsule_net_udp/src/server/handlers/mod.rs:17` to
`userland/capsule_net_udp/src/server/handlers/mod.rs:21`,
`userland/capsule_net_tcp/src/server/handlers/mod.rs:17` to
`userland/capsule_net_tcp/src/server/handlers/mod.rs:25`,
`userland/capsule_net_sockets/src/server/handlers/mod.rs:17` to
`userland/capsule_net_sockets/src/server/handlers/mod.rs:31`).

## 4. Payload limits

L2 reserves an IPC payload maximum of Ethernet frame maximum plus 64 bytes
(`userland/capsule_net_l2/src/protocol/limits.rs:25`). IP uses IPv4 MTU plus
64 bytes (`userland/capsule_net_ip/src/protocol/limits.rs:23`). UDP uses a
1472-byte UDP payload maximum and an IPC payload maximum of payload plus 64
bytes (`userland/capsule_net_udp/src/protocol/limits.rs:18`,
`userland/capsule_net_udp/src/protocol/limits.rs:23`). TCP uses a 1460-byte
segment payload maximum and an IPC payload maximum of segment plus 64 bytes
(`userland/capsule_net_tcp/src/protocol/limits.rs:17`,
`userland/capsule_net_tcp/src/protocol/limits.rs:18`).

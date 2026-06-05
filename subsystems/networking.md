# Networking

This page describes the userland network capsule stack under
`userland/capsule_net_*` and the NIC driver capsules it sits above. Read
[Userland Model](../userland/README.md) and [Syscall ABI Reference](../abi/syscalls.md)
first.

---

## 1. Spawn order

Init starts the network stack as userland capsules in this order: L2, IP, UDP,
DHCP, TCP, DNS, Nym, and sockets (`src/userspace/init/spawn_plan/network.rs:17`).
NIC drivers are separate capsules. The NIC driver group spawns e1000, rtl8139,
and rtl8169 when their features are enabled
(`src/userspace/init/spawn_plan/drivers_nic.rs:17`). The virtio display helper
spawns virtio-gpu and virtio-net when their features are enabled
(`src/userspace/init/spawn_plan/drivers_virtio_display.rs:17`).

```
  NIC driver capsule
        |
  net.l2
        |
  net.ip
     +-----+-----+
     |           |
  net.udp     net.tcp
   +---+       +---+
   |   |       |   |
 DHCP DNS     Nym
      |        |   |
      +--------+---+
               |
        net.sockets
```

UDP setup resolves `net.ip` and reads the current IPv4 config from it
(`userland/capsule_net_udp/src/setup.rs:36`). DNS setup resolves `net.udp`,
binds its local port, and caches the UDP port (`userland/capsule_net_dns/src/setup.rs:30`).
Nym setup resolves `net.tcp` and caches the TCP port
(`userland/capsule_net_nym/src/setup.rs:27`). The sockets capsule has client
modules for UDP, TCP, and Nym (`userland/capsule_net_sockets/src/clients/mod.rs:17`).

## 2. Common capsule pattern

The L2, IP, UDP, DHCP, TCP, DNS, and Nym capsules are no_std, no_main binaries.
They initialize the heap, wait for setup by retrying `setup::run`, yield between
attempts, then enter `server::run` (`userland/capsule_net_l2/src/main.rs:31`,
`userland/capsule_net_ip/src/main.rs:33`, `userland/capsule_net_udp/src/main.rs:29`).
DHCP also attempts an initial DORA lease before serving requests
(`userland/capsule_net_dhcp/src/main.rs:55`).

`net.sockets` initializes heap, waits until `state::discover()` succeeds, then
enters `server::run` (`userland/capsule_net_sockets/src/main.rs:30`).

## 3. Protocol surfaces

| Capsule | Ops |
|---------|-----|
| `net.l2` | healthcheck, get MAC, get link, send frame, poll frame, ARP resolve (`userland/capsule_net_l2/src/protocol/ops.rs:21`) |
| `net.ip` | healthcheck, get config, set config, send packet, poll packet, route add, route clear (`userland/capsule_net_ip/src/protocol/ops.rs:21`) |
| `net.udp` | healthcheck, bind, unbind, send, recv (`userland/capsule_net_udp/src/protocol/ops.rs:17`) |
| `net.dhcp` | healthcheck, lease request, lease status, lease release, lease renew (`userland/capsule_net_dhcp/src/protocol/ops.rs:17`) |
| `net.tcp` | healthcheck, listen, connect, accept, send, recv, close, shutdown (`userland/capsule_net_tcp/src/protocol/ops.rs:17`) |
| `net.dns` | healthcheck, resolve A, resolve AAAA, flush cache, set upstream (`userland/capsule_net_dns/src/protocol/ops.rs:17`) |
| `net.sockets` | socket, bind, listen, accept, connect, send, recv, close, getsockopt, setsockopt (`userland/capsule_net_sockets/src/protocol/ops.rs:17`) |
| `net.nym` | gateway, session, send, recv, cover traffic, close, topology, credential, SURB, reply, timing, authority, directory sync, status (`userland/capsule_net_nym/src/protocol/ops.rs:17`) |

## 4. NIC drivers

The virtio-net capsule initializes heap, retries setup until a driver is ready,
checks RX and TX DMA regions, and enters its server loop
(`userland/capsule_driver_virtio_net/src/main.rs:35`). The e1000 capsule
initializes heap, runs setup, brings the device up, and enters its server loop
(`userland/capsule_driver_e1000/src/main.rs:37`).

The production desktop GUI build depends on virtio-net artifacts, net L2, IP,
UDP, DHCP, TCP, DNS, sockets, and Nym artifacts before building the kernel
(`Makefile:1086`).

# Address, Names, and Anonymity

A network host needs an address, a way to resolve names, and, in NØNOS, an optional anonymity overlay.
DHCP obtains the address, DNS resolves names, and the `nym` capsule provides a mixnet path. This page
documents those three at the subsystem level. The consolidated code is under
`userland/capsule_net_core/src/iface/`; the decomposed per-layer equivalents are on the
[layers](layers.md) page, and the mixnet has its own [mixnet](mixnet.md) page.

## DHCP

In the consolidated stack, `net_core` obtains its IPv4 address by DHCP using smoltcp's DHCP socket
(`userland/capsule_net_core/src/iface/dhcp/`). On bring-up the stack starts a DHCP client, which
discovers a server, requests a lease, and installs the assigned address, gateway, and DNS server into
the interface. This is the piece that was proven at runtime: on a desktop-GUI boot the stack completes
driver setup and reaches a bound lease, `DHCP BOUND 10.0.2.15` under QEMU's user network, which is the
end-to-end evidence that the driver capsule, the frame path, the smoltcp interface, and the DHCP client
all work together. The lease is renewed by the same client as it ages. In the decomposed stack the same
job is a standalone `net_dhcp` capsule running the DORA state machine over `net.l2` and `net.ip`
([layers](layers.md)).

## DNS

Name resolution is a DNS resolver in `net_core` (`src/protocol/dns.rs`), using the DNS server the DHCP
lease provided. A client asks the [sockets service](sockets.md) or `net_core` to resolve a name, and
the resolver issues a DNS query over UDP through the same stack and returns the address. Because the
resolver runs in the network capsule, name resolution is subject to the same capability boundary as the
rest of the stack: a capsule that cannot reach the network cannot resolve names either. The decomposed
stack splits this into a standalone `net_dns` capsule that queries over `net.udp` and takes its
upstream address from the DHCP lease ([layers](layers.md)).

## The nym mixnet

`nym` (`userland/capsule_net_nym/`) is the anonymity overlay. Where the base stack carries traffic
directly, the nym capsule wraps an application datagram in its own layered Sphinx packet and routes it
through a mixnet, so the correspondence between a capsule's traffic and its destination is obscured. It
is a separate capsule reached only by IPC to `net.tcp`, present when the build includes it and absent
otherwise; traffic that does not use it takes the direct path.

It is not Tor and not a VPN. A route home is three mix layers plus the gateway holding the session,
four Sphinx hops, with the entry gateway handed the packet directly as a fifth node. Single-use reply
blocks let an exit answer without learning the origin, and the signed node directory is fetched over
the stack's own [TLS 1.3](tls.md). The full packet format, the per-hop cryptography, the reply blocks,
the directory sync and its retry hardening, and the failure modes are on the [mixnet](mixnet.md) page.

## Source

```
  userland/capsule_net_core/src/iface/dhcp/     the consolidated DHCP client
  userland/capsule_net_core/src/protocol/dns.rs  the consolidated DNS resolver
  userland/capsule_net_nym/                      the mixnet overlay capsule (see mixnet.md)
```

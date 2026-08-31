# The Node Directory

This page documents the signed node directory: the `NYMD` wire format, the Ed25519 authority check, the
validity window and epoch anti-rollback, the node record parse, the layered four-hop route selection, and the
TLS fetch that pulls a directory over `net.tcp`. It mirrors `src/topology/` and `src/directory_sync/`. The
route header that consumes the selected hops is on the [mixnet](mixnet.md) page; the trusted-authority store
the verify path consults is on the [state](state.md) page.

## Why a directory

A mixnet client cannot route without knowing the nodes, their addresses, their published X25519 packet keys,
and their roles. The directory is that list, and because a lying directory would deanonymize every user, it is
signed. The capsule stores exactly one directory at a time and will not route until a valid one is installed
(`src/topology/store.rs:46`). Installing a directory resets every open session, because the routes those
sessions would take have changed underneath them (`src/server/handlers/set_topology.rs:32`).

## The NYMD wire format

A directory is a 128-byte header followed by fixed-size node records (`src/topology/types.rs:17`):

```
  offset  size  field                              types.rs / parse.rs / layout.rs
  0       4     magic = "NYMD"                      DIR_MAGIC:17, parse.rs:46
  4       1     version = 1                         DIR_VERSION:18, parse.rs:49
  5       1     reserved, zero                      parse.rs:52
  6       2     node count                          layout.rs:25
  8       8     epoch                               parse.rs (meta):67
  16      8     not-before (ms)                     parse.rs:55
  24      8     not-after (ms)                      parse.rs:56
  32      32    issuer public key                   parse.rs:64
  64      64    Ed25519 signature                   verify.rs:31
  128     ..    node records, 76 bytes each         NODE_WIRE_LEN:7
```

`layout::check_len` requires the body to be at least the 128-byte header, reads the node count, rejects zero
(`Empty`) and a count over `NODE_CAP` (128) (`TooLarge`), and requires the body length to be exactly
`128 + count * 76` (`src/topology/layout.rs:21`). Each node record is 76 bytes:

```
  offset  size  field                     node.rs
  0       1     role (1 entry, 2 mix, 3 exit)
  1       1     layer
  2       2     delay_ms
  4       4     IP
  8       2     port     (routing / mix port a header names)
  10      32    identity
  42      32    packet_key (X25519 public)
  74      2     ws_port  (WebSocket port a client dials)
```

`node::parse` reads those fields and rejects an unknown role byte (`src/topology/node.rs:3`). A node carries
both ports because a gateway answers on a different port for client sessions than for the packets it routes,
so a routing address cannot double as a dial address (`src/topology/types.rs:26`).

## The signature check

`install` runs three checks in order before it stores anything (`src/topology/parse.rs:25`): the length check
above, a header check, and the signature check. The header check requires the `NYMD` magic, version 1, a zero
reserved byte, and a coherent validity window, `not_after > not_before` and the current clock inside
`[not_before, not_after)` (`src/topology/parse.rs:45`). The clock comes from `mk_time_millis`, and a negative
read is a `Clock` error (`src/topology/clock.rs:21`).

The signature check is in `src/topology/verify.rs:23`. The signed message is the first 64 bytes of the header
(everything up to the signature) concatenated with the node records, so the signature covers the metadata and
the node list but not the signature field itself (`src/topology/layout.rs:41`). The issuer key is the 32 bytes
at offset 32. Before verifying, the code asks the trusted-authority store whether that issuer is the trusted
signer: `Some(true)` proceeds, `Some(false)` is `UntrustedAuthority`, and `None`, meaning no authority is set
at all, is `NoAuthority` (`verify.rs:26`). Only then does it call `crypto_ed25519_verify`, returning
`BadSignature` on a non-zero result (`verify.rs:32`). This is fail-closed: with no authority installed no
directory verifies, and a directory signed by anyone but the installed authority is rejected.

## Epoch anti-rollback and freshness

`store::replace` holds the parsed directory behind a mutex and enforces two more rules on install
(`src/topology/store.rs:31`). It re-checks freshness against the clock, and it rejects a directory whose epoch
is less than or equal to the stored one, so an attacker cannot replay an older, still-in-window directory to
force a stale node set (`store.rs:39`). On read, `snapshot` re-checks that the stored directory is still
inside its validity window and that its issuer is still the trusted authority before handing out the node
list, so a directory that expires or an authority that is rotated away invalidates routing immediately
without a separate revocation step (`store.rs:46`). The status reported by `OP_TOPOLOGY_STATUS` comes from
the same predicates: `Missing`, `Ready`, `Expired`, `Clock`, or `UntrustedAuthority` (`src/topology/status.rs:20`).

## Route selection

`route` selects four hops from the current directory using the route seed (`src/topology/select.rs:29`). The
route is fixed shape: three mixes at layers 1, 2, and 3, then an exit gateway:

```
  hop 0  Mix layer 1     select.rs:32
  hop 1  Mix layer 2     select.rs:33
  hop 2  Mix layer 3     select.rs:34
  hop 3  ExitGateway     select.rs:35
```

Five nodes carry a packet, but only these four are hops the header holds a layer for. The **entry gateway is
not a Sphinx hop**: the capsule already holds a session with it, hands it the packet directly, and it forwards
to the first mix the header names (`select.rs:22`, `ROUTE_HOPS = 4` at `src/topology/types.rs:11`). Listing
the entry gateway as hop 0 would ask it to forward the packet to itself. The two route variants that pin the
last hop, `route_to` (ending at the recipient's gateway) and `route_home` (ending at ours), are on the
[mixnet subsystem](../../subsystems/networking/mixnet.md) page.

For each position, `pick` filters the node set to the matching role and, for mixes, the matching layer, then
selects one deterministically by taking a seed byte modulo the number of candidates (`select.rs:39`). An empty
candidate set at any position is `MissingHop`, which becomes `E_NO_ROUTE` at the send site. The selection is
deterministic given the seed, so the same session and payload reproduce the same route, while different
sessions spread across the available nodes. `snapshot` is what enforces that selection only ever runs against
a fresh, trusted directory (`select.rs:30`).

## Two ways a directory is installed

There are two install paths, with two different trust models. Both feed the same `store`, freshness, and epoch
rules above once the node set is in hand.

**Signed push (`OP_SET_TOPOLOGY`, `OP_SYNC_DIRECTORY`).** A `NYMD` body is verified through the Ed25519
`install` path: the transport is untrusted and the operator signature is what is trusted. `OP_SYNC_DIRECTORY`
can fetch that body rather than take it in the request; the source is a bounded IPv4 address, port, host, and
path (`src/directory_sync/source.rs:30`), and a source given once is remembered so a later empty-body sync
reuses it (`src/server/handlers/sync_directory.rs`).

**Live TLS fetch (`directory_tick`).** The path that actually runs at boot pulls the current node list from
the real Nym validator API over the stack's own [TLS 1.3](../../subsystems/networking/tls.md), not plain HTTP.
`live::fetch_role` calls `fetch_tls(tcp_port, "validator.nymtech.net", path)` and parses the JSON node objects
the API returns (`src/directory_sync/live.rs:60`, `:9`), across three separate fetches, one per role:
mixnodes, entry gateways, and exit gateways (`live.rs:14`). This directory is authenticated only by the TLS
certificate chain, which is weaker than an operator-pinned signature, so it is recorded as `fetched` rather
than `Signed` (`live.rs:23`). It is installed through `install_fetched`, still subject to the freshness and
epoch store.

The fetch is deliberately hardened against boot-time crypto contention, because each TLS handshake can lose
its certificate hash to a busy crypto pool and come back as a certificate error (net/nym code 20). The three
stages retry: the gateway and exit fetches each re-ask up to twelve times (`src/directory_sync/stages.rs:49`,
`:82`), and the directory tick is not considered done until the installed directory carries **both** a gateway
and an exit (`src/server/directory_tick.rs:56`). An earlier revision fetched the gateway list once with no
retry; an empty gateway list made a route home impossible and every send was refused, and stopping on a
gateways-but-no-exit sync left the mixnet unusable in a way that never recovered. After a successful sync,
`rebind_if_unknown` drops the boot-time gateway if the fresh directory does not describe it and the serve loop
dials one it does (`src/server/rebind.rs:34`). The full sequence is on the
[mixnet subsystem](../../subsystems/networking/mixnet.md) page.

## Real versus design

The verify, epoch, freshness, and selection logic is real and self-contained: it makes a genuine
`crypto_ed25519_verify` call, enforces anti-rollback, and produces a concrete four-hop route. What the tree
does not contain is a NONOS-run directory authority or a NONOS-operated set of mix nodes. The live path
therefore points at the public Nym validator and trusts the TLS chain; a `NYMD`-signing operator can be
supplied through `OP_SET_TOPOLOGY` for the stronger signed model. The capsule is the client and verifier of a
directory; it is not the authority that issues one.

## Source map

```
  userland/capsule_net_nym/src/topology/types.rs      DIR_MAGIC, NODE_WIRE_LEN, ROUTE_HOPS, Node, Role
  userland/capsule_net_nym/src/topology/parse.rs      install: length, header, signature, node parse
  userland/capsule_net_nym/src/topology/layout.rs     the length check and the signed-message assembly
  userland/capsule_net_nym/src/topology/verify.rs     the trusted-authority gate and Ed25519 verify
  userland/capsule_net_nym/src/topology/node.rs       the 74-byte node record parse
  userland/capsule_net_nym/src/topology/store.rs      the epoch anti-rollback and freshness store
  userland/capsule_net_nym/src/topology/select.rs     the layered four-hop route selection
  userland/capsule_net_nym/src/topology/status.rs     the OP_TOPOLOGY_STATUS predicates
  userland/capsule_net_nym/src/topology/clock.rs      the mk_time_millis wrapper
  userland/capsule_net_nym/src/directory_sync/live.rs       the live TLS fetch of the Nym validator API
  userland/capsule_net_nym/src/directory_sync/stages.rs     the three-stage fetch with gateway/exit retry
  userland/capsule_net_nym/src/directory_sync/https.rs      the TLS fetch over net.tcp
  userland/capsule_net_nym/src/directory_sync/source.rs     the signed-push source parse
  userland/capsule_net_nym/src/server/directory_tick.rs     the idle-tick sync, done only with gateway+exit
  userland/capsule_net_nym/src/server/rebind.rs             move onto a directory-described gateway
```

Every reference above is verified against those trees.

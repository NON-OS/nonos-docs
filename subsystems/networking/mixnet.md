# The Nym Mixnet

NØNOS ships its own Sphinx mixnet. It is not Tor, not a VPN, and not a wrapper around an external
Nym library: the packet format, the per-hop cryptography, the route construction, the reply blocks,
the signed directory, and the cover traffic are all hand-written no_std code inside one userland
capsule, `userland/capsule_net_nym/`, that reaches the wire only by IPC to `net.tcp`. This page
documents the live mixnet path against the source, with `file:line` references, and is honest about
where the implementation simplifies canonical Sphinx and where it depends on infrastructure the tree
does not contain.

For where the capsule sits and its capability mask, see the [nym capsule reference](../../userland/net-nym/README.md).
The base IP stack it does not use is the [layered stack](layers.md); the TLS the directory sync runs
over is [TLS 1.3](tls.md).

## Two packet formats, one op

`OP_SEND` (op 4) carries a session's bytes into the mixnet, and it branches on whether the session
has been bound to a Nym destination (`src/server/handlers/send.rs:54`):

- If a destination is set (`OP_SET_DESTINATION`, op 17), the send is sealed as a full **Sphinx**
  packet through `send_sphinx` (`send.rs:55`, `src/server/handlers/send_sphinx.rs:28`). This is the
  live path the browser and the SOCKS5 exit use, and it is what the rest of this page documents. It
  carries its own per-hop authentication and needs no credential.
- Otherwise the send falls back to an older custom `NYMP` packet format
  (`packet::encode`, `send.rs:64`, `src/packet/` and `src/route/`), which is credential-gated. The
  comment at `send.rs:52` is explicit that the credential belongs to that older format. That path is
  still compiled but is not the one browser traffic takes; its 365-byte header and five-block layout
  are documented on the [nym mixnet](../../userland/net-nym/mixnet.md) and
  [nym packet](../../userland/net-nym/packet.md) pages and are not repeated here.

## Sphinx packet format

The wire packet is fixed size regardless of payload, so length leaks nothing. The sizes are defined
and compile-time asserted in `src/sphinx/constants/sizes.rs:65`:

```
  k  (security parameter)     16 bytes   fields.rs:18
  r  (max path length)         5 hops    fields.rs:21
  node meta info              44 bytes   sizes.rs:21  (addr 32 + flag 1 + delay 8 + version 3)
  encrypted routing info     300 bytes   sizes.rs:29  (44+16) * 5
  header                     348 bytes   sizes.rs:37  (32 group element + 16 MAC + 300 routing)
  payload overhead            17 bytes   sizes.rs:38  (k + 1)
  regular payload           2065 bytes   sizes.rs:42  (2048 plaintext + 17)
  regular packet            2413 bytes   sizes.rs:43  (348 header + 2065 payload)
```

The header holds `MAX_PATH_LENGTH = 5` routing slots even though a route uses four (below); the extra
slot is what the stream cipher refills as each hop strips the slot it consumed, so the header length
never changes as the packet travels (`sizes.rs:34`, `sizes.rs:54`). The const assertions in
`sizes.rs:65-74` exist because an earlier revision split the 2413-byte total wrong and every mix
silently rejected the packet; the sizes are now checked at compile time.

## Per-hop cryptography

Each hop's key material comes from one Diffie-Hellman result stretched to 288 bytes and cut into five
pieces (`src/sphinx/keys/mod.rs:17`, widths in `src/sphinx/constants/fields.rs:31`):

```
  shared_i  = X25519(ephemeral_scalar, node_i.packet_key)    crypto/ecdh.rs:30 (crypto_x25519_shared syscall)
  expanded  = HKDF-SHA256(shared_i)  -> 288 bytes            constants/kdf.rs, sizes.rs:45
              stream cipher key      16   fields.rs:35
              integrity MAC key      16   fields.rs:32
              payload key           192   fields.rs:40   (LIONESS: two ChaCha20 + two BLAKE2b keys)
              blinding factor        32   fields.rs:33
              replay tag             32   fields.rs:34
```

- **Key agreement** is X25519, drawn from the kernel `crypto_x25519_*` syscalls
  (`src/crypto/ecdh.rs:21`, `:30`). A single fresh ephemeral scalar is used per packet; the group
  element is blinded between hops with the per-hop blinding factor, the canonical Sphinx construction
  (`src/sphinx/keys/blinding_factor.rs`).
- **Key expansion** is HKDF-SHA256 with the network's fixed `info` string (the Archimedes lever
  quote, `constants/kdf.rs:19`); one changed byte of that string yields different keys at every hop.
- **Payload encryption** is LIONESS, a wide-block cipher instantiated over ChaCha20 and BLAKE2b
  (`src/crypto/lioness/mod.rs:17`). A payload is one block however long it is, so flipping one
  ciphertext bit scrambles the whole plaintext, which is what denies a mix the ability to tag a
  packet by a payload bit and recognise it downstream. One LIONESS layer is laid per hop, outermost
  last, so the first mix peels its layer and hands on something it cannot read (`src/sphinx/payload/seal.rs:22`).
- **Header integrity** is a per-hop 16-byte MAC (`src/sphinx/mac/`), verified before a hop acts on
  its routing block, so a tampered header is dropped rather than forwarded.
- **Replay defence** is a 64-entry ring of replay tags; a tag already seen is refused
  (`src/state/replay.rs:19`, `:32`).

The ChaCha20, BLAKE2b, LIONESS, GCM-SIV, and Polyval primitives are implemented inside the capsule
(`src/crypto/`); Polyval carries its own test vectors (`src/crypto/polyval/vectors.rs`). X25519,
HKDF-SHA256, HMAC, BLAKE3, Ed25519-verify, and random are kernel `crypto_*` syscalls reached through
the capsule's `Crypto` capability.

## Route construction: four hops

`topology::route` selects the route from the signed directory and the route seed
(`src/topology/select.rs:29`). It returns exactly four nodes:

```
  hop 0   Mix, layer 1     select.rs:32
  hop 1   Mix, layer 2     select.rs:33
  hop 2   Mix, layer 3     select.rs:34
  hop 3   ExitGateway      select.rs:35
```

Five nodes carry a packet, but only four are hops the header holds a layer for. The **entry gateway
is the fifth node and is not a Sphinx hop**: the capsule already holds a WebSocket session with it,
hands it the packet directly, and it forwards to the first mix the header names (`select.rs:22`,
`src/topology/types.rs:8`, `ROUTE_HOPS = 4` at `types.rs:11`). Listing the entry gateway as hop 0
would ask it to forward the packet to itself.

Per position, `pick` filters the directory to the matching role and mix layer and selects one
deterministically by a seed byte modulo the candidate count (`select.rs:39`). An empty candidate set
at any position is `MissingHop`, which surfaces as `E_NO_ROUTE`. The route is drawn fresh per packet,
not per message (`src/mixnet/seal.rs:29`): two packets of one message share no path, so a mix that
sees both cannot group them.

Two route variants pin the last hop, because only one specific gateway can hand a packet to a given
client:

- `route_to` ends the forward route at the gateway the **recipient** registered with
  (`src/mixnet/route_to.rs:32`). Ending anywhere else delivers to a node that has never heard of the
  recipient, which still answers the acknowledgement and then drops the message: every packet
  answered, nothing arrives (`route_to.rs:24`).
- `route_home` ends the reply and acknowledgement route at **our own** gateway
  (`src/mixnet/route_home.rs:33`). It returns nothing until the directory describes our gateway,
  because routing to it needs the address and packet key the directory publishes, which cannot be
  guessed (`route_home.rs:29`).

If no directory has arrived yet, `sphinx_route` falls back to the compiled-in bootstrap operator
nodes so the mixnet is reachable at all (`src/mixnet/route.rs:27`).

## SURBs: answering without an address

An exit never learns where the client is, so it cannot address a reply. A **single-use reply block**
is the only way it can answer (`src/surb/build.rs:24`). `build_surb` constructs a Sphinx header for a
route that ends at us, draws a fresh random symmetric key, and returns the header, its first-hop
address, and the per-hop payload keys (`surb/build.rs:33`). The reply is sealed with that key before
the route is applied, and the client is the only party that kept a copy. The destination in the block
is our identity; the identifier field is unused by this network and sent as zeros (`build.rs:47`).

A request that wants an answer is a **repliable message**: the message type, a sender tag, a content
tag, the SURB count, the SURBs, then the request (`src/message/repliable.rs:30`). Without a SURB
attached the far end has no route back at all. A recipient keeps a reserve of SURBs it will not spend
and says so rather than going quiet when it runs low; the answer is a message that carries nothing but
more reply blocks (`repliable_additional_surbs`, `repliable.rs:53`). The number issued per request is
`SURBS_PER_REQUEST` (`src/surb/supply.rs`).

## Acknowledgements and fragmentation

A message is padded so its size says nothing, then split across as many fixed-size packets as it
needs (`src/mixnet/encode_message.rs:52`, `src/message/prepare.rs`). Each fragment carries its own
SURB-acknowledgement built over the route home (`encode_message.rs:85`, `src/ack/`) and its own key
agreement, so two packets of one message share nothing an observer could group them by. The exit
lifts the acknowledgement out of the payload and sends it home, which is how the client learns a
fragment landed. Acknowledgement packets travel narrower than messages, and the width is how a hop
tells the two apart (`src/sphinx/constants/sizes.rs:84`).

## Delay and cover traffic

The mixing itself is per-hop delay: packets that arrive together leave apart, so an observer at both
ends cannot pair them (`src/mixnet/delays.rs:20`). The per-hop delay is taken from the selected
node's `delay_ms` in the directory (`delays.rs:31`); because node selection is randomised per packet,
the delays differ per packet, but note honestly that the delay is the node's configured value rather
than a per-packet exponential sample as in canonical Nym.

Cover traffic keeps an idle session from being silent. `OP_COVER_TICK` (op 6) emits a burst of random
`FLAG_COVER` packets on a jittered timer (`src/server/handlers/cover.rs`). The policy is a burst count
clamped to 1..8 and a jitter clamped to 10..10000 ms, set by `OP_SET_TIMING` (op 12) and sampled with
kernel randomness (`src/state/timing.rs:31`, `:76`).

## Directory sync over TLS

The mixnet cannot route without the node list: addresses, published X25519 packet keys, roles, and
per-hop delays. That list is the signed `NYMD` directory, fetched over the stack's own
[TLS 1.3](tls.md) rather than pushed. `directory_tick` runs the fetch on an idle tick because it
takes a round trip and a handshake, and a capsule that stops answering while it waits looks dead
(`src/server/directory_tick.rs:47`).

The fetch is three separate TLS requests, one list at a time in the order a route needs them
(`src/directory_sync/stages.rs`):

1. **Mix layers** (`stages.rs:26`). A list missing a layer cannot carry a route, so a fetch that
   comes back without all layers is `Failed(13)` rather than installed.
2. **Gateways** (`stages.rs:56`). Retried up to `GATEWAY_ATTEMPTS = 12` (`stages.rs:49`).
3. **Exits** (`stages.rs:89`). Retried up to `EXIT_ATTEMPTS = 12` (`stages.rs:82`), then installed.

The retries matter because each TLS fetch can lose its certificate hash to a busy crypto pool early
in boot and come back as a certificate error (net/nym code 20; see [TLS 1.3](tls.md)). The mix and
gateway lists already in hand prove the host and the chain are fine, so a later failure is transient
contention, not a broken endpoint.

The hardening that made this reliable is threefold and is worth stating precisely, because an earlier
revision got it wrong:

- **Both retries.** The exit fetch was always retried; the gateway fetch was a single attempt. An
  empty gateway list is worse than an empty exit list, because a route home ends at the gateway
  holding our session, so with no gateway the directory describes, no reply block can be built and
  every send is refused. The gateway fetch is now retried the same way (`stages.rs:40-69`).
- **Not done until both present.** `directory_tick` returns early only when
  `directory_gateway_count() > 0` **and** `directory_exit_count() > 0`
  (`directory_tick.rs:56`, counts in `src/state/directory_gateway.rs:50`, `:60`). A first sync can
  land one list and not the other; stopping on either alone left the mixnet unusable in a way that
  never recovered. On a sync that arrives missing one, the outcome handler schedules a short resync
  rather than hammering the API (`src/server/directory_outcome.rs:49`), with a doubling backoff
  capped at `MAX_BACKOFF = 4` on hard failures (`directory_tick.rs:35`, `directory_outcome.rs:55`).
- **Rebind onto a described gateway.** The first gateway is dialled from the compiled list before any
  directory exists, so it is very likely not in the one that arrives. After a successful sync
  `rebind_if_unknown` drops the session if our gateway is not in the directory, and the serve loop
  redials one the directory does describe (`src/server/rebind.rs:34`, called from
  `directory_outcome.rs:41`). It does nothing if the directory has no gateways, so it never puts the
  capsule back on the compiled list it just left (`rebind.rs:41`).

### Directory verification

A fetched directory is verified exactly like a pushed one; the transport is untrusted and the
signature is what is trusted. `install` runs a length check, a header check (magic `NYMD`, version 1,
a coherent validity window against `mk_time_millis`), then the signature check
(`src/topology/parse.rs`, `src/topology/layout.rs`). Verification is fail-closed: the issuer key must
be the installed trusted authority (`Some(true)`), an unknown issuer is `NoAuthority` and a
non-matching one is `UntrustedAuthority`, and only then is `crypto_ed25519_verify` called
(`src/topology/verify.rs:23`). The store enforces epoch anti-rollback, rejecting a directory whose
epoch is not greater than the stored one, and re-checks the validity window and trusted authority on
every read, so an expired directory or a rotated authority invalidates routing with no separate
revocation step (`src/topology/store.rs`). The node record is 76 bytes and carries both ports a node
answers on, the routing/mix port and the WebSocket port a client dials, plus the identity and X25519
packet key (`src/topology/types.rs:5`, `:20`).

## Transport to the gateway

The outermost packet leaves over a link the capsule speaks itself: an RFC 6455 WebSocket to the entry
gateway, carried as a byte stream over `net.tcp` (`src/gateway_client/ws/`, `src/gateway_client/mod.rs`).
`make_encrypted_blob` frames the Sphinx packet as a `KIND_FORWARD_SPHINX` blob for the gateway
(`gateway_client/mod.rs`, `src/server/handlers/send_sphinx.rs:51`). The capsule holds no NIC and no
driver authority; it reaches the wire only by IPC to `net.tcp`.

## Failure modes

| Condition | Where | Result |
|---|---|---|
| `net.tcp` not up | `send.rs:44` | `E_NO_TCP` |
| A mix layer or the exit gateway has no candidate | `select.rs:47` | `MissingHop` -> `E_NO_ROUTE` |
| Our / the recipient's gateway not yet in the directory | `route_home.rs:34`, `route_to.rs:33` | route is `None`, send refused |
| Directory TLS cert hash lost to a busy crypto pool | `stages.rs:74` | net/nym code 20, retried up to 12x |
| Directory arrives with gateways but no exit (or vice versa) | `directory_tick.rs:56` | short resync scheduled, tick keeps running |
| Replayed packet | `replay.rs:32` | dropped |
| No credential (legacy `NYMP` path only) | `send.rs:57` | `E_NO_CREDENTIAL` and friends |

## Status

| Mechanism | Status |
|---|---|
| Sphinx packet sizing (2413 B, 348 B header) | IMPLEMENTED, compile-time asserted (`sizes.rs:65`) |
| X25519 per-hop key agreement, HKDF-SHA256 expansion | IMPLEMENTED |
| LIONESS payload cipher (ChaCha20 + BLAKE2b) | IMPLEMENTED; Polyval has reference vectors |
| Four-hop route (3 mix + exit gateway) + entry gateway | IMPLEMENTED, ENFORCED (`select.rs:29`) |
| SURBs and repliable messages | IMPLEMENTED |
| Per-fragment acknowledgements, fragmentation, padding | IMPLEMENTED |
| Replay window (64 tags) | IMPLEMENTED, ENFORCED |
| Signed NYMD directory: Ed25519, epoch anti-rollback, validity window | IMPLEMENTED, ENFORCED (fail-closed) |
| Directory sync over TLS with gateway+exit retry hardening | IMPLEMENTED, ENFORCED |
| Cover traffic and jittered timing | IMPLEMENTED |
| End-to-end under QEMU (gateway handshake, directory, 2-hop SOCKS5) | PROVEN under QEMU |
| End-to-end on real hardware | NOT PROVEN in this tree |
| NØNOS-run directory authority / live mixnodes | NOT IN TREE: the capsule is the client and verifier, not the authority (`../../userland/net-nym/directory.md`); routes depend on an external operator and `validator.nymtech.net` |
| Per-packet exponential delay sampling | NOT IMPLEMENTED: delay is the selected node's configured `delay_ms` |

## Source

```
  userland/capsule_net_nym/src/sphinx/           the live Sphinx packet, header, keys, MAC, payload
  userland/capsule_net_nym/src/mixnet/           route_to, route_home, seal, delays, encode_message
  userland/capsule_net_nym/src/surb/             single-use reply block construction
  userland/capsule_net_nym/src/message/          repliable messages, fragmentation, padding
  userland/capsule_net_nym/src/topology/         NYMD directory parse, verify, epoch store, route select
  userland/capsule_net_nym/src/directory_sync/   the three-stage TLS fetch and retry stages
  userland/capsule_net_nym/src/server/           directory_tick, directory_outcome, rebind, send handlers
  userland/capsule_net_nym/src/gateway_client/   the WebSocket link to the entry gateway over net.tcp
  userland/capsule_net_nym/src/crypto/           LIONESS, ChaCha20, BLAKE2b, GCM-SIV, Polyval, X25519, HKDF
  userland/capsule_net_nym/src/state/replay.rs   the 64-tag replay window
```

# capsule_wallet_nonos

`capsule_wallet_nonos` is the most substantial application in the tree: an Ethereum and NOX wallet that
signs transactions through the keyring and talks to a JSON-RPC endpoint over a **from-scratch TLS 1.3
client it implements itself**, 197 source files and roughly 7,000 lines, with no networking crate and no
TLS library. It draws its own UI, holds no private keys of its own (the keyring does), and verifies the
server certificate chain against a single pinned root. This page documents it exhaustively. App
`app.wallet_nonos`, an [app-skeleton](../writing-an-app.md) GUI app. The source is
`userland/capsule_wallet_nonos/`.

## Contents

- [The shape of the capsule](#the-shape-of-the-capsule)
- [State](#state)
- [The four views and the event model](#the-four-views-and-the-event-model)
- [Keys and addresses: the keyring boundary](#keys-and-addresses-the-keyring-boundary)
- [Building and signing a transaction](#building-and-signing-a-transaction)
- [JSON-RPC](#json-rpc)
- [The TLS 1.3 client](#the-tls-13-client)
- [The certificate trust model](#the-certificate-trust-model)
- [The network path](#the-network-path)
- [Security analysis](#security-analysis)
- [Debugging](#debugging)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The shape of the capsule

The entry (`src/main.rs:27`) is a single line, `run(wallet::Wallet::new)`: the wallet is an app-skeleton
GUI application, so the [runtime](../writing-an-app.md) owns the surface, the window, the input
subscription, and the paint loop, and the wallet supplies an `App` implementation. Everything else hangs
off `wallet/` (`src/wallet/mod.rs`): `state` (the model), `event` (input handling), `paint` (rendering),
`ipc` (the keyring calls), `rpc` (JSON-RPC construction and parsing), `net` (sockets and probing),
`tls13` (the TLS client), and `tx_hash` (Keccac-256 of a transaction). The capsule holds no server; it is
a client of the keyring and the network.

## State

The whole wallet is one `State` struct (`src/wallet/state/types.rs:37`), and every field is worth
enumerating because the UI and the flows are entirely a function of it:

```
  keyring_port, owner_pid, wallet_id      the keyring binding and this wallet's id
  address[20], address_ready              the Ethereum address, and whether it is loaded
  balance_wei[32], balance_ready          the balance as a 256-bit big-endian wei value
  live_nonce, nonce_ready                 the account nonce from the chain
  fee_wei, fee_ready                      the current fee estimate
  view                                    VIEW_HOME | RECEIVE | SEND | PROOF
  send_focus, send_to_hex[40], send_to_len   the Send form: which field, the recipient hex
  send_amount_milli_eth, send_nonce       the amount (in milli-ETH) and the nonce to sign
  tx_hash[32], tx_raw, tx_len, tx_ready    the constructed transaction and its hash
  tx_kind                                  a label ("eth" / "nox" / "none")
  broadcast_ready, broadcast_hash[32]      the broadcast result
  receipt_ready, receipt_ok               the receipt poll result
  proof_count, proof_eth_hash[32], proof_nox_hash[32]   the signed-transaction proofs
  net: NetStatus                          the network probe status
  rails[8], rail_count                    the settlement rails (see below)
  status                                  a status line
```

The starting state (`src/wallet/state/new.rs:18`) is `view = HOME`, `send_amount_milli_eth = 1`, and
`status = "keyring pending"`, everything else zero or not-ready. A `Rail` (`types.rs:27`) is a settlement
rail, a symbol, a family, a status, a chain id, and a 20-byte contract; the wallet supports up to eight,
and it reads the allowed set from [policy](policy.md) (`state/filter_rails.rs`) so a build governs which
chains are offered.

## The four views and the event model

The UI is four views selected by `state.view`: **Home** (the balance and address), **Receive** (the
address to receive to), **Send** (the recipient, amount, and nonce form), and **Proofs** (the signed
transaction hashes). Input arrives through the app skeleton's `on_event`
(`src/wallet/event/on_event.rs`), which splits into `on_key` and `on_pointer`. Key handling drives the
Send form: `edit_amount`, `edit_nonce`, `recipient`, and `hex_digit` build up the form fields, and the
action keys trigger `generate` (a new wallet), `sign_eth` / `sign_nox` / `sign_both`, `broadcast`, and
`probe_net`. Each action mutates the state and requests a repaint. The `paint/` tree renders the current
view from the state; it is large (a header, sidebar, tabs, account card, network card, rail cards, and a
proof view) but purely a projection of `State`.

## Keys and addresses: the keyring boundary

The wallet never holds a private key. Its cryptographic operations are IPC calls to the
[keyring](keyring.md) (`src/wallet/ipc/`):

- `lookup_keyring` resolves the keyring service port, and `address` fetches this wallet's Ethereum
  address (`OP_WALLET_ADDRESS`).
- `generate` creates a new wallet in the keyring (`OP_WALLET_GENERATE`).
- `sign_eth` and `sign_nox` request signatures (`OP_SIGN_ETH_TRANSFER`, the NOX ops).
- `decode_rails` / `read_rails` enumerate the wallet rails (`OP_LIST_WALLET_RAILS`).

The private key lives in the keyring, which wipes it after each signing (see the [keyring](keyring.md)
page), so a compromise of the wallet capsule exposes the UI and the network path but not the signing key.

## Building and signing a transaction

An Ethereum transfer is built and signed in one keyring call (`src/wallet/ipc/sign_eth.rs:23`). The
wallet marshals an EIP-1559 transfer with fixed gas parameters and hands it to the keyring, which holds
the key, does the secp256k1 signing, and returns the signed raw transaction:

```
  sign_eth_transfer(port, owner_pid, wallet_id, to[20], nonce, value_wei):
      payload = owner_pid || wallet_id || to
              || word(nonce)
              || word(1_500_000_000)      // maxPriorityFeePerGas = 1.5 gwei
              || word(30_000_000_000)     // maxFeePerGas         = 30 gwei
              || word(21_000)             // gasLimit             = 21000
              || word(value_wei)
      rx = keyring_call(port, OP_SIGN_ETH_TRANSFER, payload)
      return rx[HDR_LEN..]                 // the signed raw transaction
```

Each `word` is a 128-bit big-endian field (`ipc/push_word.rs`). The gas parameters are fixed constants
appropriate for a plain transfer (a 21,000-gas send at 1.5/30 gwei). The wallet then computes the
transaction hash itself with Keccak-256 over the raw bytes (`src/wallet/tx_hash.rs:17`, a
`crypto_keccak256` syscall), storing it as `tx_hash` and a proof. The NOX path
(`ipc/sign_nox.rs`) is analogous through the keyring's NOX signing ops.

## JSON-RPC

The wallet speaks Ethereum JSON-RPC, which it constructs and parses by hand (`src/wallet/rpc/`). Each
request is a small function that builds the JSON body; `request_broadcast`
(`src/wallet/rpc/request_broadcast.rs:19`) is representative:

```
  request_broadcast(raw_tx, id):
      {"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0x<hex(raw_tx)>"],"id":<id>}
```

The set covers `request_nonce` (`eth_getTransactionCount`), `request_balance` (`eth_getBalance`),
`request_fee`, `request_chain_id` (`eth_chainId`), `request_broadcast` (`eth_sendRawTransaction`), and
`request_receipt` (`eth_getTransactionReceipt`). The responses are parsed by dedicated parsers,
`parse_quantity32` (a `0x`-prefixed hex quantity into a 32-byte big-endian value), `parse_hash32` (a
32-byte hash), `parse_u64`, and `parse_receipt_ok`, and `http_post` / `http_body` wrap a request in a
minimal HTTP/1.1 POST and extract the response body. There is no JSON crate; the wallet writes and reads
the exact byte sequences it needs.

## The TLS 1.3 client

The wallet's defining feature is that it implements TLS 1.3 from scratch (`src/wallet/tls13/`, roughly a
hundred files) rather than depending on a TLS library. It offers one cipher suite and one key-exchange
group (`src/wallet/tls13/constants.rs`):

```
  TLS 1.3 (0x0304)   ChaCha20-Poly1305-SHA256 (0x1303)   X25519 (0x001d)
```

**The key schedule** (`src/wallet/tls13/schedule.rs:24`) is the standard TLS 1.3 HKDF derivation, and it
is textbook-correct:

```
  early     = HKDF-Extract(salt=0, ikm=0)
  derived   = Derive-Secret(early, "derived", SHA-256(""))     // EMPTY_HASH is the SHA-256 of empty
  handshake = HKDF-Extract(salt=derived, ikm=ECDHE_shared)     // the X25519 shared secret
  th        = SHA-256(transcript)
  client_hs = Derive-Secret(handshake, "c hs traffic", th)
  server_hs = Derive-Secret(handshake, "s hs traffic", th)
  key       = HKDF-Expand-Label(secret, "key", "")             // per direction
  iv        = HKDF-Expand-Label(secret, "iv", "")
```

The `EMPTY_HASH` constant is the SHA-256 of the empty string (`schedule.rs:19`), exactly as RFC 8446
requires, and the labels (`"derived"`, `"c hs traffic"`, `"s hs traffic"`, `"key"`, `"iv"`) are the
standard ones. `expand_label` (`tls13/expand_label.rs`) implements HKDF-Expand-Label over the wallet's
own HKDF and SHA-256/384.

**The record layer** (`src/wallet/tls13/record_seal.rs:19`) is the TLS 1.3 AEAD record, and it is exact:

```
  seal(key, iv, seq, inner_type, body):
      plaintext = body || inner_type                    // the TLSInnerPlaintext content type suffix
      header    = 0x17 || 0x0303 || (len(plaintext) + 16)   // application_data, legacy version, +tag
      nonce     = iv XOR big_endian(seq)                // per-record nonce
      ct        = ChaCha20-Poly1305(key, nonce, aad=header, plaintext)
      return header || ct
```

The nonce construction (`tls13/nonce.rs:17`) XORs the big-endian record sequence number into the low
eight bytes of the static IV, per RFC 8446, and the record header is used as the AEAD additional data.
`record_open.rs` is the inverse. The handshake is driven by a set of "flight" functions
(`client_flight`, `server_first_flight`, `server_certificate_flight`, `server_finished_flight`, and so
on) that assemble the ClientHello with its SNI, supported-versions, supported-groups, key-share, and
signature-algorithms extensions, and parse the server's ServerHello, EncryptedExtensions, Certificate,
CertificateVerify, and Finished. `finished_verify.rs` checks the server Finished MAC and
`client_finished.rs` produces the client's.

## The certificate trust model

The wallet does not carry a general root store; it pins a single anchor. `chain_anchor`
(`src/wallet/tls13/chain_anchor.rs:17`) requires the server to present at least two certificates, takes
the last one (the root or a self-issued anchor), extracts its SubjectPublicKeyInfo, SHA-256-hashes it,
and requires that hash to equal a hardcoded constant (`gts_r4_anchor.rs:17`):

```
  chain_anchor(certs):
      require cert_count >= 2
      anchor = last cert
      require SHA-256(SPKI(anchor)) == GTS_R4         // Google Trust Services R4, pinned by SPKI hash
```

`GTS_R4` is the 32-byte SHA-256 of the Google Trust Services R4 root's public-key info. The leaf and
intermediate signatures are verified up the chain: `verify_leaf` (`tls13/verify_leaf.rs:17`) extracts the
leaf's to-be-signed bytes and signature, takes the issuer's P-256 public point from its SPKI, SHA-256s
the TBS, converts the DER signature to a raw 64-byte `(r, s)`, and calls `verify_p256`;
`verify_intermediate` does the same up the chain, with `verify_p384` available for P-384 issuers. The
hostname is checked against the certificate's subject-alternative-name dNSNames
(`cert_dns_match.rs:17`), including single-label wildcards (`*.example.com` matches one label), and the
validity window is checked against the current time (`cert_valid_now.rs:17`, parsing both `UTCTime`
`0x17` and `GeneralizedTime` `0x18` notBefore/notAfter and requiring `from <= now <= until`). So the
chain is verified for signature, name, validity, and a pinned root.

## The network path

Under TLS is the socket layer (`src/wallet/net/`), which uses the [net.sockets](../../subsystems/networking/sockets.md)
service: `socket_open`, `socket_connect`, `socket_send`, `socket_recv`, and `socket_close` are the
transport, `read_tls_flight` reads a TLS record flight off the socket, and the `probe_*` functions
(`probe_chain_id`, `probe_account`, `probe_status`, `probe_tls_rpc`, `probe_rpc_tcp`) test reachability
and pull the account state. `resolve_eth` resolves the RPC host, and `rtc_stamp` reads the current time
for certificate validity. So a balance refresh is: resolve the host, open and connect a socket through
`net.sockets`, run the TLS 1.3 handshake (ClientHello through Finished, verifying the chain), seal an
HTTP POST carrying the JSON-RPC `eth_getBalance` into a TLS record, read and open the response records,
and parse the hex quantity, all in the capsule.

## Security analysis

The wallet's trust boundaries are worth stating precisely:

- **Keys** never leave the [keyring](keyring.md). The wallet requests signatures and receives signed
  transactions; it cannot exfiltrate a private key because it never has one.
- **The RPC channel** is authenticated by a full TLS 1.3 handshake with certificate-chain verification:
  leaf and intermediate ECDSA signatures, hostname match, validity window, and a pinned root anchor. A
  man-in-the-middle without a chain to the pinned GTS R4 root is rejected.
- **The capability boundary**: the wallet holds only the capabilities its manifest declared (IPC to the
  keyring and the network, memory), so a compromise is contained to what those allow.

The honest caveat on the trust anchor is that it is a *single pinned root* (GTS R4), not a general CA
store: this is a deliberate, conservative choice, it means the wallet trusts exactly one certificate
authority hierarchy and rejects everything else, which is stronger than a broad root store for a wallet
talking to a known endpoint, but it also means the RPC endpoint must chain to that specific root. The
TLS client offers one suite (ChaCha20-Poly1305) and one group (X25519), which is a small, auditable
surface rather than a full negotiation matrix.

## Debugging

The wallet fails in layers, and it is instrumented so the layer that failed is visible rather than
hidden behind a blank window. The order to check them in follows the network path from the outside in.

The first check is that the capsule spawned. The wallet is brought up through
`super::boot::capsule("APP-NONOS-WALLET", ...)` (`src/userspace/init/spawn_plan/apps.rs:74`), which prints
`[APP-NONOS-WALLET] capsule spawned` on success and a spawn error on failure. If that line is absent the
ELF failed verification or the manifest asked for a capability outside policy, and none of the wallet's
own code ran.

If the window is up but the account never loads, the next stop is the keyring, because the address, the
balance nonce, and every signature are IPC calls to it. The starting `status` line is `"keyring pending"`
(`src/wallet/state/new.rs:18`), and a wallet stuck on that string never resolved the keyring port: the
[keyring](keyring.md) capsule is down or the wallet was not granted IPC to it. The address and balance
fields carry their own `_ready` flags in `State` (`address_ready`, `balance_ready`, `nonce_ready`,
`fee_ready`), so the UI shows which of the four account facts arrived and which is still missing, which
separates a keyring problem (no address) from a network problem (address but no balance).

If the address loads but the balance and nonce do not, the failure is in the network path, and the
`probe_*` functions exist to localise it before you touch the signing code. `probe_network`
(`src/wallet/net/probe.rs:22`) runs the ladder in order and folds the results into `NetStatus`:
`probe_rpc_tcp` (`net/probe_rpc_tcp.rs:24`) reports `resolve`, `socket`, and `connect` as three separate
booleans, so a DNS failure, a socket-open failure, and a TCP-connect failure are distinguishable rather
than one opaque timeout; then `probe_tls_rpc` (`net/probe_tls_rpc.rs:39`) attempts the full TLS 1.3
handshake and `probe_status` (`net/probe_status.rs:17`) turns the combination into the status string the
Home view shows. So a wallet that resolves and connects but shows no balance is failing inside the TLS
handshake or the certificate chain, not at the socket, and the reverse narrows it to the network stack
below TLS. The certificate path itself is the sharpest failure: `chain_anchor`
(`src/wallet/tls13/chain_anchor.rs:17`) rejects a server whose chain does not terminate in the pinned
GTS R4 root, and `cert_valid_now.rs` rejects one outside its validity window, so an endpoint that swaps
its CA or a clock that is wrong (the validity check reads `rtc_stamp`) presents as a handshake that
completes on the wire but is refused by the wallet. That is a deliberate rejection, not a bug, and it is
the one to suspect when the same endpoint worked before and does not now.

Signing is the last layer and the easiest to confirm, because it produces a proof. A successful
`sign_eth` or `sign_nox` stores the transaction hash in `proof_eth_hash` / `proof_nox_hash` and bumps
`proof_count` (`src/wallet/state/types.rs`), rendered in the Proofs view, so a signature that the keyring
performed is visible as a hash without broadcasting anything. If the Proofs view fills but a broadcast
never lands, the split is clean: the keyring signed (proof present) and the failure is the network send,
which is the same `probe_*` ladder again.

## Honest gaps

Documented plainly rather than glossed: the gas parameters in the EIP-1559 transfer are fixed constants
(1.5/30 gwei, 21,000 gas) rather than derived per-transaction from the fee estimate, suitable for a plain
transfer but not for contract calls; the wallet needs the keyring and policy ports at runtime and
degrades if they are absent; network probes can time out; and the TLS client implements the subset of TLS
1.3 it needs (one suite, one group, the certificate and finished handshake) rather than the full
protocol (no session resumption, no client certificates, no post-handshake authentication). The rails
offered are whatever policy allows, so the set is build- and configuration-governed.

## Source map

```
  src/main.rs                         run(Wallet::new)
  src/wallet/state/                   State (types.rs), new.rs, hydrate.rs, filter_rails.rs, record_tx.rs
  src/wallet/event/                   on_event -> on_key / on_pointer; sign_eth/nox/both, broadcast, generate
  src/wallet/paint/                   ~40 files: header, sidebar, tabs, account/network/rail cards, proofs
  src/wallet/ipc/                     the keyring calls: address, generate, sign_eth, sign_nox, rails
  src/wallet/tx_hash.rs               Keccak-256 of the raw transaction
  src/wallet/rpc/                     JSON-RPC request builders and response parsers, http_post/http_body
  src/wallet/net/                     sockets over net.sockets, TLS flight reads, the probe_* functions
  src/wallet/tls13/                   the TLS 1.3 client (~100 files):
      constants.rs                        suite, group, extension ids
      schedule.rs, expand_label.rs, hkdf.rs, hash_sha256/384.rs   the key schedule
      record_seal.rs, record_open.rs, nonce.rs, aad_frame.rs      the AEAD record layer
      client_hello.rs, ext_*.rs, client_flight.rs                 the ClientHello
      server_*_flight.rs, server_hello.rs, scan_*.rs              parsing the server flights
      chain_anchor.rs, gts_r4_anchor.rs                           the pinned-root check
      verify_leaf.rs, verify_intermediate.rs, verify_p256/384.rs  chain signature verification
      cert_dns_match.rs, cert_valid_now.rs, cert_tbs/spki/signature.rs   name, validity, field extraction
      finished_verify.rs, finished_value.rs, client_finished.rs   the Finished MACs
```

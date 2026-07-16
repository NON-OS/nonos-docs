# capsule_wallet_nonos (full reference)

`capsule_wallet_nonos` is the wallet application in the NONOS tree: a GUI window that generates and
addresses an Ethereum/NOX account, signs transactions, and talks to a public Ethereum JSON-RPC endpoint
over a TLS 1.3 client it implements itself. It holds no private key of its own. Every signature is an IPC
call to the [keyring](keyring.md), which owns the key material; the wallet marshals the transaction, gets
the signed raw bytes back, and can broadcast them. The network path is roughly a hundred files of a
from-scratch TLS 1.3 client with no TLS library and no networking crate, verifying the server chain
against a single pinned root.

It is an [app-skeleton](../writing-an-app.md) GUI app. The kernel spawns it under service handle
`app.nonos_wallet` on service port 4734 with a reply port on 4735, and its capability mask is `0x1839`
(`userland/capsule_wallet_nonos/Capsule.mk:10`). The source is `userland/capsule_wallet_nonos/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [User reference](#user-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [The key and signing path](#the-key-and-signing-path)
- [Protocol and IPC](#protocol-and-ipc)
- [The TLS 1.3 client and trust model](#the-tls-13-client-and-trust-model)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The wallet is an ordinary NONOS GUI application. Its entry point is a single line, `run(Wallet::new)`, so
the runtime owns the surface, the window, the input subscription, and the paint loop, and the wallet
supplies an `App` implementation (`src/main.rs:28`, `src/wallet/app.rs:35`). The `App` gives the runtime
three things: a manifest for a normal window (`src/wallet/manifest.rs:26`), an `on_event` that turns
keystrokes and pointer clicks into view changes, form edits, and actions (`src/wallet/app.rs:40`), and a
`paint` that draws the current view (`src/wallet/app.rs:48`).

Behind the window the whole wallet is one `State` struct (`src/wallet/state/types.rs:37`). There is no
private key in it. The account address, the balance, the nonce, the fee, the constructed transaction, and
the signed-transaction proofs are all facts fetched from the keyring or the chain and cached in `State`,
and the four views are a projection of it. The capsule holds no server of its own; it is a client of the
keyring and of the network stack.

## Identity

Everything the kernel and the service registry need to name and reach the wallet comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `wallet-nonos` | `Capsule.mk:1` |
| Service handle | `app.nonos_wallet` | `Capsule.mk:2`, `src/userspace/capsule_wallet_nonos/spawn.rs:29` |
| Namespace | `systems.nonos.app.nonos_wallet` | `Capsule.mk:7` |
| Service endpoint | `service:4734:app.nonos_wallet` | `Capsule.mk:8`, `spawn.rs:30` |
| Reply endpoint | `reply:4735:endpoint.app.nonos_wallet.reply` | `Capsule.mk:9`, `spawn.rs:31`, `spawn.rs:32` |
| Capability mask | `0x1839` | `Capsule.mk:10` |
| Binary name | `wallet_nonos` | `Capsule.mk:5` |
| Kernel mirror | `src/userspace/capsule_wallet_nonos` | `Capsule.mk:11` |

The mask `0x1839` decomposes bit by bit against `src/capabilities/types.rs`, and the kernel spawn path
requests exactly these six capabilities and no others (`spawn.rs:48`):

```
  0x0001  CoreExec                bit()    1     types.rs:56
  0x0008  IPC                     bit()    8     types.rs:59
  0x0010  Memory                  bit()   16     types.rs:60
  0x0020  Crypto                  bit()   32     types.rs:61
  0x0800  GraphicsDisplayQuery    bit() 2048     types.rs:67
  0x1000  GraphicsSurfaceCreate   bit() 4096     types.rs:68
  ------
  0x1839  = 1 + 8 + 16 + 32 + 2048 + 4096
```

There is no `Network` bit (4) and no `FileSystem` bit (64) in the mask, which is the basis of the security
analysis below: the wallet can execute, ask the display for its size, create a surface, speak IPC, and use
the kernel crypto primitives (Keccak and AEAD, used for the transaction hash and the TLS record layer),
but it cannot open a socket or touch a device on its own. Every RPC packet it appears to send is really an
IPC request to the `net.sockets` service that holds the real transport authority, and every signature is
an IPC request to the keyring that holds the key.

## User reference

The window has four views selected by `state.view`, and every action is a key or a pointer click. Key
handling is a flat set of single-key shortcuts in `on_key` (`src/wallet/event/on_key.rs:21`); when the
Send view is active, most non-shortcut keys are diverted into the Send form
(`on_key.rs:36`, `src/wallet/event/send_input.rs:21`). The pointer handler hit-tests the sidebar,
buttons, and Send fields (`src/wallet/event/on_pointer.rs:21`). This is honest about what is a real
end-to-end operation and what only exercises the signing and hashing path.

### Navigation and views

| Action | Key / click | Effect | Handler |
|---|---|---|---|
| Home view | `h` or `1`, or sidebar row 1 | show the account card, balance, and network card | `on_key.rs:27`, `on_key.rs:31`, `on_pointer.rs:46` |
| Receive view | `v` or `2`, or sidebar row 2 | show the address to receive to | `on_key.rs:28`, `on_key.rs:32`, `on_pointer.rs:46` |
| Send view | `s` or `3`, or sidebar row 3 | show the recipient, amount, and nonce form | `on_key.rs:29`, `on_key.rs:33`, `on_pointer.rs:46` |
| Proofs view | `p` or `4`, or sidebar row 4 | show the signed-transaction hashes | `on_key.rs:30`, `on_key.rs:34`, `on_pointer.rs:46` |
| Close the window | Esc | end the app | `src/wallet/event/on_event.rs:23` |

Note `s` selects the Send view only while another view is active; once Send is focused, `s` is a printable
key the form ignores (it is not a hex digit), so it does not toggle back. Use `h`, `1`, or a sidebar click
to leave Send.

### Account actions

| Action | Key / click | Effect | Handler |
|---|---|---|---|
| Generate a wallet | `g` / `G`, or the Generate button | ask the keyring to create a new wallet, fetch its address, switch to Receive | `on_key.rs:35`, `on_pointer.rs:31`, `src/wallet/event/generate.rs:22` |
| Refresh / hydrate | `r` / `R` | re-run the keyring and rail hydrate step | `on_key.rs:22`, `src/wallet/state/hydrate.rs:22` |
| Probe the network | `w` / `W`, or the Probe button | run the DNS/socket/TLS/chain probe ladder and, if the address is loaded, pull balance/nonce/fee | `on_key.rs:41`, `on_pointer.rs:33`, `src/wallet/event/probe_net.rs:22` |

`generate` fails cleanly if the keyring is unavailable: the status line becomes `generate failed` or
`address failed` and no wallet id is set (`generate.rs:33`, `generate.rs:38`). A refresh with no keyring
port leaves the status at `keyring unavailable` (`hydrate.rs:29`).

### The Send form

The Send form has three fields cycled with Tab and edited in place (`send_input.rs:21`). It is only active
in the Send view.

| Action | Key / click | Effect | Handler |
|---|---|---|---|
| Cycle field | Tab | move focus To -> Amount -> Nonce -> To | `send_input.rs:22` |
| Focus a field | click the field row | set focus to To, Amount, or Nonce | `on_pointer.rs:55` |
| Edit recipient | hex digit `0-9 a-f A-F` while To is focused | append one hex nibble, up to 40 (a 20-byte address) | `send_input.rs:33`, `src/wallet/event/recipient.rs:19` |
| Edit amount | digit while Amount is focused | build the amount in milli-ETH | `src/wallet/event/edit_amount.rs` |
| Edit nonce | digit while Nonce is focused | build the nonce | `src/wallet/event/edit_nonce.rs` |
| Backspace | Backspace | drop the last recipient nibble, or divide amount/nonce by ten | `send_input.rs:40` |
| Sign and stay | Enter, or the Sign button in Send | sign the ETH transfer | `send_input.rs:26`, `on_pointer.rs:61` |

The amount is entered in milli-ETH and converted to wei by multiplying by `1_000_000_000_000_000`, with
an overflow check that reports `amount too large` rather than wrapping (`src/wallet/event/eth_value.rs:17`,
`src/wallet/event/sign_eth.rs:31`). The recipient must be exactly 40 hex characters or `recipient` returns
nothing and signing reports `recipient incomplete` (`recipient.rs:20`, `sign_eth.rs:27`). The starting
amount is `1` milli-ETH (`src/wallet/state/new.rs:35`).

### Signing and broadcast

| Action | Key | Effect | Handler |
|---|---|---|---|
| Sign ETH transfer | `E` (or Enter in Send) | build an EIP-1559 transfer, sign it through the keyring, hash it, store it as a proof, switch to Proofs | `on_key.rs:37`, `sign_eth.rs:22` |
| Sign NOX approve | `n` / `N` | request a NOX approval signature from the keyring, hash it, store it as a proof | `on_key.rs:38`, `src/wallet/event/sign_nox.rs:22` |
| Sign both | `P` | sign an ETH transfer and a NOX approval in one action, record both proofs, set `proof_count = 2` | `on_key.rs:39`, `src/wallet/event/sign_both.rs:23` |
| Broadcast | `b` / `B`, or the Broadcast button in Proofs | send the stored signed transaction to the RPC and poll one receipt | `on_key.rs:40`, `on_pointer.rs:39`, `src/wallet/event/broadcast.rs:21` |

All three signing actions require a wallet to exist first; without one the status is `generate wallet
first` (`sign_eth.rs:23`, `sign_nox.rs:23`, `sign_both.rs:24`). A successful sign stores the raw
transaction and its Keccak-256 hash in `State` via `record_tx`, sets the last-signed transaction as the
broadcast candidate, and switches to the Proofs view (`src/wallet/event/sign_result.rs:24`,
`src/wallet/state/record_tx.rs:19`). `sign_both` records the NOX transaction as the broadcast candidate
and both hashes as proofs (`sign_both.rs:61`). Broadcast is honest about being end-to-end: it refuses with
`no signed tx` if nothing is staged, sends the raw bytes through `net.sockets`, and folds the receipt poll
into `receipt confirmed`, `receipt pending`, `broadcast sent`, or `broadcast rejected`
(`broadcast.rs:22`, `broadcast.rs:37`).

Real versus demonstration, stated plainly. Address generation, signing, and the transaction hash are real:
they cross into the keyring and the kernel Keccak primitive and return genuine bytes, and the Proofs view
shows the real hash of a real signed transaction. Balance, nonce, and fee are real reads from a live chain
when the network is reachable and the address is loaded (`probe_net.rs:24`). The broadcast is a real
`eth_sendRawTransaction` when the network path is up. The gas parameters in the EIP-1559 transfer are
fixed constants rather than derived from the live fee estimate (see below), so a send is a plain 21,000-gas
transfer, not a general contract call; the NOX path signs a fixed approve template rather than a
user-composed call (`sign_nox.rs:27`). The Send form drives the ETH path end to end; the NOX and
sign-both actions produce proofs but sign fixed templates.

## Architecture and lifecycle

The capsule is `no_std`/`no_main` and hands its `App` to the skeleton (`src/main.rs:17`, `:28`). The
`wallet/` module tree splits into `state` (the model), `event` (input handling), `paint` (rendering),
`ipc` (the keyring calls), `rpc` (JSON-RPC construction and parsing), `net` (sockets and probing),
`tls13` (the TLS client), `tx_hash` (Keccak-256 of a transaction), `hex`, `theme`, and `manifest`
(`src/wallet/mod.rs:17`).

Lifecycle:

1. The kernel spawns the capsule at boot through the desktop-fleet plan (`src/userspace/init/spawn_plan/apps.rs:23`,
   `:72`), which calls `super::boot::capsule("APP-NONOS-WALLET", ...)` (`apps.rs:75`). That verifies the
   embedded ELF, id cert, manifest, and attestation trailer, registers `app.nonos_wallet` on port 4734,
   and logs `[APP-NONOS-WALLET] capsule spawned` on success (`spawn.rs:35`,
   `src/userspace/init/capsule_boot/run.rs:29`).
2. The skeleton `run` creates the window from the manifest: a `WIDTH x HEIGHT` Normal window titled
   `NONOS Wallet` at `(370, 128)`, subscribing to key-down, absolute-pointer, and button-down input
   (`src/wallet/manifest.rs:26`). The input mask is `KeyDown | PointerAbs | ButtonDown`
   (`manifest.rs:21`).
3. On the first event or the first tick, `hydrate` runs once: it resolves the keyring port and this
   wallet's own pid through service lookups, then reads and filters the settlement rails
   (`src/wallet/app.rs:41`, `:52`, `hydrate.rs:22`).
4. Each event flows through `on_event` to `on_key` or `on_pointer`, which mutate `State` and return
   `Repaint`, `Idle`, or `Close` (`on_event.rs:21`). `paint` then projects the active view: background,
   sidebar, topbar, one of the four view bodies, and the status bar (`src/wallet/paint/paint.rs:21`). The
   frame lands in the shared surface the compositor presents.

The paint tree is large but purely a projection of `State`: a header, sidebar, tabs, an account card, a
network card, rail cards, and the proof view, spread across roughly forty files under `src/wallet/paint/`.

## The key and signing path

The wallet never holds a private key. Its cryptographic operations are IPC calls to the
[keyring](keyring.md) service, resolved by name and reached through a small request/reply helper
(`src/wallet/ipc/lookup_keyring.rs:21`, `src/wallet/ipc/call.rs:23`). The keyring op numbers are fixed in
one place (`src/wallet/ipc/constants.rs`):

```
  OP_WALLET_GENERATE      9    create a new wallet, returns a wallet id     constants.rs:19
  OP_WALLET_ADDRESS      10    fetch a wallet's 20-byte address             constants.rs:20
  OP_SIGN_NOX_APPROVE    12    sign a NOX approval, returns raw tx          constants.rs:21
  OP_SIGN_ETH_TRANSFER   13    sign an EIP-1559 transfer, returns raw tx    constants.rs:22
  OP_LIST_WALLET_RAILS   14    enumerate the wallet's settlement rails      constants.rs:23
```

An Ethereum transfer is built and signed in one keyring call (`src/wallet/ipc/sign_eth.rs:23`). The wallet
marshals the transfer fields and hands them to the keyring, which holds the key, does the secp256k1
signing, and returns the signed raw transaction after an 8-byte reply header (`sign_eth.rs:40`,
`call.rs:23`):

```
  sign_eth_transfer(port, owner_pid, wallet_id, to[20], nonce, value_wei):
      payload = le(owner_pid) || le(wallet_id) || to[20]
              || word(nonce)
              || word(1_500_000_000)      // maxPriorityFeePerGas = 1.5 gwei
              || word(30_000_000_000)     // maxFeePerGas         = 30 gwei
              || word(21_000)             // gasLimit             = 21000
              || word(value_wei)
      rx = keyring_call(port, OP_SIGN_ETH_TRANSFER, payload)
      return rx[HDR_LEN..]                 // the signed raw transaction
```

Each `word` is a 32-byte EVM word carrying a 128-bit value big-endian in its low 16 bytes
(`src/wallet/ipc/push_word.rs:19`). The gas parameters are fixed constants appropriate for a plain
21,000-gas transfer at 1.5/30 gwei (`sign_eth.rs:36`). The NOX path signs a fixed approve template
(85,000 gas, a 1 ETH value word) through `OP_SIGN_NOX_APPROVE` (`src/wallet/ipc/sign_nox.rs:23`). After
either call the wallet computes the transaction hash itself with Keccak-256 over the raw bytes through a
kernel crypto syscall (`src/wallet/tx_hash.rs:17`), storing it as `tx_hash` and as a proof. The private
key lives in the keyring, so a compromise of the wallet capsule exposes the UI and the network path but
not the signing key.

## Protocol and IPC

The wallet registers no application opcodes of its own beyond the `app.nonos_wallet` service and reply
inbox the skeleton registers for it (`spawn.rs:39`). Everything else it does is an outbound IPC call.

Keyring, service `keyring`, resolved by name; each request is `seq(4) | op(2) | pad(2) | payload` and each
reply is `seq(4) | status(4) | body`, with a nonzero status surfaced as an error (`call.rs:23`). The ops
are the five above; the calls are `generate_wallet`, `wallet_address`, `sign_eth_transfer`,
`sign_nox_approve`, and `read_rails` (`src/wallet/ipc/`). Rails are decoded and then filtered to the set
the wallet recognises, `ETH`, `NOX`, and `PR` (`src/wallet/ipc/decode_rails.rs:20`,
`src/wallet/state/rail_allowed.rs:19`, `src/wallet/state/filter_rails.rs:21`).

JSON-RPC, constructed and parsed by hand with no JSON crate (`src/wallet/rpc/`). Each request is a small
function that writes the exact byte sequence; `request_broadcast` is representative
(`src/wallet/rpc/request_broadcast.rs:19`):

```
  {"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0x<hex(raw_tx)>"],"id":<id>}
```

The set covers `request_nonce` (`eth_getTransactionCount`), `request_balance` (`eth_getBalance`),
`request_fee`, `request_chain_id` (`eth_chainId`), `request_broadcast` (`eth_sendRawTransaction`), and
`request_receipt` (`eth_getTransactionReceipt`) (`src/wallet/rpc/`). Responses are parsed by dedicated
parsers: `parse_quantity32` (a `0x` hex quantity into a 32-byte big-endian value), `parse_hash32`,
`parse_u64`, and `parse_receipt_ok`. `http_post` wraps the JSON in a minimal HTTP/1.1 POST with a `Host`,
`Content-Length`, and `Connection: close` (`src/wallet/rpc/http_post.rs:19`), and `http_body` extracts the
response body.

Network transport, service `net.sockets`, magic `0x4E534B54`, with `net.dns` for resolution
(`src/wallet/net/constants.rs:17`). The socket ops are `OP_SOCKET 2`, `OP_CONNECT 6`, `OP_SEND 7`,
`OP_RECV 8`, `OP_CLOSE 9` (`constants.rs:22`), wrapped by `socket_open`, `socket_connect`, `socket_send`,
`socket_recv`, and `socket_close` (`src/wallet/net/`). The RPC host is `ethereum-rpc.publicnode.com`
(`constants.rs:18`), resolved through `net.dns` `OP_RESOLVE_A 2` (`constants.rs:21`). A `net.nym` port is
probed for health but the RPC path itself does not route through it (`src/wallet/net/probe.rs:25`). So a
balance refresh is: resolve the host, open and connect a socket on port 443 through `net.sockets`, run the
TLS 1.3 handshake verifying the chain, seal an HTTP POST carrying the JSON-RPC into a TLS record, read and
open the response records, and parse the hex quantity, all inside the capsule.

## The TLS 1.3 client and trust model

The wallet's defining feature is that it implements TLS 1.3 from scratch (`src/wallet/tls13/`, roughly a
hundred files) rather than depending on a TLS library. It offers one cipher suite and one key-exchange
group (`src/wallet/tls13/constants.rs`):

```
  TLS 1.3 (0x0304)   ChaCha20-Poly1305-SHA256 (0x1303)   X25519 (0x001d)
```

The key schedule (`src/wallet/tls13/schedule.rs:24`) is the standard TLS 1.3 HKDF derivation and is
textbook-correct: `early = HKDF-Extract(0, 0)`, `derived = Derive-Secret(early, "derived", EMPTY_HASH)`,
`handshake = HKDF-Extract(derived, ECDHE_shared)`, then `c hs traffic` and `s hs traffic` secrets from the
transcript hash, and `key` / `iv` per direction via HKDF-Expand-Label. `EMPTY_HASH` is the SHA-256 of the
empty string, exactly as RFC 8446 requires (`schedule.rs:19`). The record layer (`src/wallet/tls13/record_seal.rs:19`)
is the TLS 1.3 AEAD record: it appends the inner content type, builds the `0x17 || 0x0303 || len+16`
header, XORs the record sequence number into the static IV to form the nonce, and seals with
ChaCha20-Poly1305 using the header as additional data (through the kernel AEAD primitive, which is why the
mask carries the Crypto bit). `record_open.rs` is the inverse.

The trust model pins a single anchor rather than carrying a general root store. `chain_anchor`
(`src/wallet/tls13/chain_anchor.rs:17`) requires at least two certificates, takes the last one, extracts
its SubjectPublicKeyInfo, SHA-256-hashes it, and requires that hash to equal a hardcoded constant, the
Google Trust Services R4 root pinned by SPKI hash (`src/wallet/tls13/gts_r4_anchor.rs:17`). The leaf and
intermediate signatures are verified up the chain: `verify_leaf` extracts the to-be-signed bytes and the
signature, takes the issuer's P-256 point from its SPKI, hashes the TBS, converts the DER signature to a
raw `(r, s)`, and calls `verify_p256`, with `verify_p384` available for P-384 issuers
(`src/wallet/tls13/verify_leaf.rs`, `verify_intermediate.rs`, `verify_p256.rs`, `verify_p384.rs`). The
hostname is checked against the certificate SAN dNSNames including single-label wildcards
(`src/wallet/tls13/cert_dns_match.rs`), and the validity window is checked against the current time, parsed
from the RTC (`src/wallet/tls13/cert_valid_now.rs`, `src/wallet/net/rtc_stamp.rs:19`). So the chain is
verified for signature, name, validity, and a pinned root.

## Security analysis

The wallet's trust boundaries are worth stating precisely.

- Keys never leave the [keyring](keyring.md). The wallet requests signatures and receives signed
  transactions; it cannot exfiltrate a private key because it never has one (`src/wallet/ipc/sign_eth.rs`,
  `sign_nox.rs`). The address, generation, and rail enumeration are the same IPC boundary.
- The RPC channel is authenticated by a full TLS 1.3 handshake with certificate-chain verification: leaf
  and intermediate ECDSA signatures, hostname match, validity window, and a pinned root anchor
  (`src/wallet/tls13/chain_anchor.rs:17`). A man-in-the-middle without a chain to the pinned GTS R4 root is
  rejected.
- The capability boundary is the mask `0x1839`: CoreExec, IPC, Memory, Crypto, GraphicsDisplayQuery, and
  GraphicsSurfaceCreate, and nothing else (`Capsule.mk:10`, `spawn.rs:48`). There is no Network bit and no
  FileSystem bit, so the wallet cannot open a socket or a file directly. The transport it uses is an IPC
  call to `net.sockets`, which holds the real authority; a bug in the wallet's RPC or TLS code cannot
  escalate past what that service already permits for this pid. The Crypto bit is present because the
  wallet uses the kernel Keccak and AEAD primitives (transaction hashing and the TLS record layer), not
  because it holds any key.
- Isolation from other capsules is the kernel's, not the wallet's: it is a CPL 3 user binary that speaks
  IPC and owns its surface, verified and enrolled at spawn like every other capsule.

Honest gaps, documented rather than glossed. The trust anchor is a single pinned root (GTS R4), not a
general CA store: a deliberate, conservative choice for a wallet talking to a known endpoint, but it means
the RPC endpoint must chain to that specific root, and swapping CA or a wrong clock presents as a handshake
that completes on the wire but is refused by the wallet. The TLS client offers one suite and one group, a
small auditable surface with no resumption, no client certificates, and no post-handshake authentication.
The EIP-1559 gas parameters are fixed constants (1.5/30 gwei, 21,000 gas) rather than derived from the live
fee estimate, so the ETH send suits a plain transfer, not a contract call, and the NOX action signs a fixed
approve template rather than a user-composed call (`src/wallet/ipc/sign_nox.rs:27`). Balance, nonce, fee,
and broadcast all need the network path up and degrade to status lines when it is not. The rails offered
are whatever the keyring returns filtered to `ETH`, `NOX`, and `PR` (`rail_allowed.rs:19`).

## How to contribute

The source lives at `userland/capsule_wallet_nonos/`. The model is under `src/wallet/state/`, the input
handlers under `src/wallet/event/`, the renderer under `src/wallet/paint/`, the keyring calls under
`src/wallet/ipc/`, the JSON-RPC under `src/wallet/rpc/`, the socket and probe layer under `src/wallet/net/`,
and the TLS 1.3 client under `src/wallet/tls13/`.

To add a user action:

1. Write the handler as one file under `src/wallet/event/`, taking `&mut State` and returning an
   `EventOutcome` (for example `src/wallet/event/generate.rs`). Report failure by setting `state.status`
   to a byte string and returning `EventOutcome::Repaint`, never by panicking.
2. Wire the key into the match in `src/wallet/event/on_key.rs:21`, or the pointer hit-test into
   `src/wallet/event/on_pointer.rs:21`, and re-export the module from `src/wallet/event/mod.rs`.
3. If it touches the keyring, add the op number to `src/wallet/ipc/constants.rs` and the marshalling to a
   new file under `src/wallet/ipc/`, following the request/reply shape in `src/wallet/ipc/call.rs`.
4. If it needs a new render, add a `paint_*` file under `src/wallet/paint/` and dispatch it from
   `src/wallet/paint/paint.rs:21`.

To build and sign the capsule, use the generated per-slug make targets. They come from the shared macro in
`nonos-mk/capsule.mk`, included through `userland/capsule_wallet_nonos/Capsule.mk:13`, which expands
`nonos-mk-<slug>` rules for `<slug> = wallet-nonos` (`nonos-mk/capsule.mk:158`):

```
  make nonos-mk-wallet-nonos               build the capsule ELF
  make nonos-mk-wallet-nonos-sign          produce the id cert, manifest, and attestation trailer
  make nonos-mk-wallet-nonos-verify        verify the signed artifacts against the trust anchor
  make nonos-mk-check-wallet-nonos-keys    check the per-capsule signing keys exist
```

There is no wallet-specific `-prod` desktop target in the Makefile; the wallet ships as part of the full
desktop image, and its verify rule is wired into the aggregate verify and artifact lists
(`Makefile:729`, `Makefile:1084`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every action reports errors as a status line and an outcome, never a panic);
modular files, one unit per file, with `mod.rs` used only for re-exports; and the AGPL header at the top of
every source file, matching the header on every existing module (`src/main.rs:1`).

## Debugging

The wallet fails in layers and is instrumented so the failing layer is visible rather than hidden behind a
blank window. Work the path from the outside in.

First confirm the capsule spawned. On a successful boot the kernel prints `[APP-NONOS-WALLET] capsule
spawned` from the boot log (`src/userspace/init/capsule_boot/run.rs:29`); the spawn path itself is
`spawn_wallet_nonos` in the desktop-fleet plan (`src/userspace/init/spawn_plan/apps.rs:72`). An absent
line means the ELF failed verification or the manifest asked for a capability outside policy, and the error
path prints an error line instead (`capsule_boot/run.rs:32`).

If the window is up but the account never loads, the next stop is the keyring, because the address, the
balance nonce, and every signature are IPC calls to it. The starting `status` line is `keyring pending`
(`src/wallet/state/new.rs:54`); hydrate turns it into `wallet ready`, `keyring unavailable`, or `rail
refresh failed` (`hydrate.rs:30`, `:36`, `:38`). A wallet stuck on `keyring pending` never ran hydrate; a
`keyring unavailable` line means the keyring port or the wallet's own pid did not resolve. The `_ready`
flags in `State` (`address_ready`, `balance_ready`, `nonce_ready`, `fee_ready`) separate a keyring problem
(no address) from a network problem (address but no balance) (`src/wallet/state/types.rs:42`).

If the address loads but balance and nonce do not, the failure is in the network path, and the probe ladder
localises it. `probe_network` runs the stages in order and folds the result into `NetStatus`
(`src/wallet/net/probe.rs:22`, `src/wallet/net/status.rs:17`): `probe_rpc_tcp` reports `resolve`, `socket`,
and `connect` as three separate booleans so a DNS failure, a socket-open failure, and a TCP-connect failure
are distinguishable (`src/wallet/net/probe_rpc_tcp.rs:24`); `probe_tls_rpc` then attempts the full TLS 1.3
handshake, and `probe_status` turns the combination into the status string the Home view shows
(`src/wallet/net/probe_status.rs:17`). That string is the sharpest diagnostic: it climbs from `route
blocked`, `route ready`, `rpc tcp ready`, `rpc tls hello`, `rpc tls record`, `rpc cert message`, `rpc cert
chain`, `rpc CA anchor`, `rpc CA signed`, `rpc host matched`, `rpc cert time`, `rpc tls finished`, `rpc
client finish`, to `rpc chain 0x1`, so the last line printed is exactly the last handshake step that
succeeded. A wallet that connects but shows no balance is failing inside TLS or the certificate chain, not
at the socket. `chain_anchor` deliberately rejects a chain that does not terminate in the pinned GTS R4
root (`src/wallet/tls13/chain_anchor.rs:17`), and `cert_valid_now` rejects one outside its validity window,
so an endpoint that swaps its CA or a wrong RTC presents as a handshake that completes on the wire but is
refused. That is a deliberate rejection, not a bug, and is the one to suspect when the same endpoint worked
before and does not now.

Signing is the last layer and the easiest to confirm because it produces a proof. A successful `sign_eth`,
`sign_nox`, or `sign_both` stores the transaction hash in `proof_eth_hash` / `proof_nox_hash` and sets
`proof_count`, rendered in the Proofs view, so a signature the keyring performed is visible as a hash
without broadcasting (`sign_both.rs:57`, `src/wallet/state/types.rs:65`). The status line reports
`transaction signed`, `2 tx proofs ready`, `transaction sign failed`, or `transaction hash failed`
(`sign_result.rs:34`, `:40`, `sign_both.rs:64`). If the Proofs view fills but a broadcast never lands, the
split is clean: the keyring signed and the failure is the network send, which is the same probe ladder
again, ending in `broadcast rejected`, `broadcast sent`, `receipt pending`, or `receipt confirmed`
(`broadcast.rs:27`, `:37`).

## Source map

```
  src/main.rs                         run(Wallet::new)
  src/wallet/app.rs                   the App impl: manifest, on_event, paint, on_tick, hydrate-once
  src/wallet/manifest.rs              the window manifest and input mask
  src/wallet/state/                   State (types.rs), new.rs, hydrate.rs, filter_rails.rs, record_tx.rs, rail_allowed.rs
  src/wallet/event/                   on_event -> on_key / on_pointer; generate, sign_eth/nox/both, broadcast, probe_net, the Send form
  src/wallet/paint/                   ~40 files: background, sidebar, topbar, home/receive/send/proof views, cards, status bar
  src/wallet/ipc/                     the keyring calls: lookup_keyring, generate, address, sign_eth, sign_nox, read_rails, decode_rails; call.rs, constants.rs, push_word.rs
  src/wallet/tx_hash.rs               Keccak-256 of the raw transaction
  src/wallet/rpc/                     JSON-RPC request builders and response parsers, http_post/http_body
  src/wallet/net/                     sockets over net.sockets, TLS flight reads, resolve_eth, rtc_stamp, the probe_* ladder, broadcast_raw, poll_receipt
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
  Capsule.mk                          slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_wallet_nonos/ the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/apps.rs   the desktop-fleet spawn entry
  src/capabilities/types.rs           the capability bit definitions the mask decomposes against
  nonos-mk/capsule.mk                 the generated nonos-mk-wallet-nonos[-sign|-verify] targets
```

Every reference above is verified against those trees.

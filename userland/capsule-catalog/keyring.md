# capsule_keyring (full reference)

`capsule_keyring` is the system's key store and the only capsule that holds private key material. It
keeps keys behind an owner-pid boundary and a per-key lock flag, imports and generates Ethereum wallets,
and signs Ethereum and NOX transactions on request. It never hands a wallet secret back to a caller: the
signers return signatures and hashes, and the retrieval path refuses to export a `Secp256k1Eth` key at
all. It is scrupulous about memory, and the exact wiping discipline is spelled out below. No other
capsule, including the [wallet](wallet-nonos.md), ever holds a private key; the wallet and the
[payment](payment.md) service reach the keyring only over IPC. The source is
`userland/capsule_keyring/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The keyring is a `no_std`/`no_main` service capsule. `_start` initializes the heap and enters the server
loop, which never returns (`userland/capsule_keyring/src/main.rs:29`). It owns a single in-memory
`Store`, receives request frames on inbox 0, dispatches each to one handler, sends a reply, and wipes the
receive buffer. There is no window, no filesystem, and no network: the keyring holds keys, answers
requests about them, and does nothing else.

Two roles sit on top of that store. First it is a general key store: any caller can `store`, `retrieve`,
`delete`, `lock`, `unlock`, and read `metadata` for its own keys, scoped strictly to the caller pid.
Second it is a wallet signer: it generates and imports secp256k1 Ethereum keys and produces EIP-1559
transactions, a NOX ERC-20 approve, and an EIP-712 payment receipt, retrieving the secret owner-checked,
signing, and zeroing the secret the instant the signature is produced. The lock flag is what [login](login.md)
toggles to gate signing behind an authenticated session, and the receipt signer is what
[payment](payment.md) calls to settle a paid request.

## Identity

Everything the kernel and the service registry need to name and reach the keyring comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `keyring` | `Capsule.mk:7` |
| Service handle | `keyring` | `Capsule.mk:8` |
| Namespace | `systems.nonos.keyring` | `Capsule.mk:13` |
| Service endpoint | `service:4098:keyring` | `Capsule.mk:14` |
| Reply endpoint | `reply:4099:endpoint.4294967298` | `Capsule.mk:15` |
| Binary name | `keyring` | `Capsule.mk:11` |
| Capability mask | `0x39` | `Capsule.mk:17` |
| Kernel mirror | `src/security/keyring_capsule` | `Capsule.mk:18` |

The reply endpoint id `4294967298` is `0x1_0000_0002`, and that is the constant the server sends every
reply to (`KERNEL_REPLY_ENDPOINT`, `src/protocol/types.rs:32`). The manifest reply port is 4099; the
kernel routes the reply frame from that endpoint id back to the caller that issued the `mk_ipc_call`.

The mask `0x39` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x08  IPC        bit()  8    types.rs:59
  0x10  Memory     bit() 16    types.rs:60
  0x20  Crypto     bit() 32    types.rs:61
  ----
  0x39  = 8 + 16 + 32
```

The comment at the head of `Capsule.mk:1` is the key point: the capsule is itself the keyring authority,
so it does *not* carry `Capability::Keyring`; callers carry that bit and reach the capsule over IPC. The
mask holds only IPC (for `mk_ipc_*`), Memory (for the heap), and Crypto (for the kernel crypto and RNG
syscall path it drives internally). There is no CoreExec bit in the mask, no Network (4), no FileSystem
(64), no Debug (256), and no Driver, Mmio, Irq, Dma, or Pio. That absence is the security property, and
the [security analysis](#security-analysis) below returns to it.

## Operation reference

The request frame is an 8-byte header, a `u32` little-endian sequence and a `u16` op followed by a `u16`
that the header layout skips over, then a payload; there is no magic (`decode_request`,
`src/protocol/decode.rs:19`, `HDR_LEN = 8` at `src/protocol/types.rs:34`). The reply is the same 8-byte
frame reused as `seq(4) || status(4)` followed by an optional body (`encode_response`,
`src/protocol/encode.rs:21`). Status `0` is success; a negative status is an errno
(`src/protocol/errno.rs`). Every operation except `list_wallet_rails` begins its payload with a four-byte
caller pid.

There are fourteen operations (`src/protocol/types.rs:17`), dispatched in `dispatch`
(`src/server/dispatch.rs:28`):

```
   1  STORE            6  METADATA        11  SIGN_NOX_RECEIPT
   2  RETRIEVE         7  COUNT           12  SIGN_NOX_APPROVE
   3  DELETE           8  WALLET_IMPORT   13  SIGN_ETH_TRANSFER
   4  LOCK             9  WALLET_GENERATE 14  LIST_WALLET_RAILS
   5  UNLOCK          10  WALLET_ADDRESS
```

An op that decode returns for but dispatch does not recognise, and any op outside 1..=14, replies
`EINVAL` (`dispatch.rs:43`).

### Caller attestation

Before any key op runs, `resolve_caller` binds the pid claimed in the payload to the pid the kernel
attested on the message (`src/server/caller.rs:17`):

```
  resolve_caller(payload_pid, sender_pid):
      if sender_pid == 0:            payload_pid    // kernel-side TCB, trusted
      if payload_pid == sender_pid:  sender_pid     // ring-3 caller must match its attested pid
      else:                          None -> EACCES
```

A ring-3 capsule can only act under its own attested pid, and every key operation is scoped to that pid,
so no capsule can retrieve, sign with, delete, lock, or read another capsule's key. A `sender_pid` of 0
is the kernel-side trusted path and is allowed to name any pid in the payload.

### Errno set

| Symbol | Value | Meaning | Source |
|---|---|---|---|
| `ENOENT` | -2 | no key with that id | `errno.rs:17` |
| `EACCES` | -13 | caller mismatch, or owner-pid check failed | `errno.rs:18` |
| `EBUSY` | -16 | key is locked | `errno.rs:19` |
| `EINVAL` | -22 | bad length, bad field, or a crypto step failed | `errno.rs:20` |
| `ENOSPC` | -28 | store is full (128 keys) | `errno.rs:21` |

### Store operations

| Op | Code | Request payload (after the 8-byte frame) | Reply body | Handler |
|---|---|---|---|---|
| `STORE` | 1 | `pid(4) || now(8) || expires(8) || key_type(1) || data_len(2) || data` | `id(4)` | `handlers/store.rs:24` |
| `RETRIEVE` | 2 | `pid(4) || id(4)` | `data` | `handlers/retrieve.rs:22` |
| `DELETE` | 3 | `pid(4) || id(4)` | empty | `handlers/delete.rs:22` |
| `LOCK` | 4 | `pid(4) || id(4)` | empty | `handlers/lock.rs:22` |
| `UNLOCK` | 5 | `pid(4) || id(4)` | empty | `handlers/unlock.rs:22` |
| `METADATA` | 6 | `pid(4) || id(4)` | 36-byte record | `handlers/metadata.rs:22` |
| `COUNT` | 7 | `pid(4)` | `count(4)` | `handlers/count.rs:22` |

`store` bounds-checks the header, resolves the caller, validates the key type through `KeyType::from_u8`
(`store/types/key_type_from_u8.rs:19`, values 0..=8), requires the frame length to match
`HDR + data_len` exactly, and inserts under the next id; the store rejects an empty or over-256-byte body
with `EINVAL` and a full store with `ENOSPC` (`store/store_key.rs:28`). The auto-incrementing id starts at
1 and wraps (`store/state.rs`, `store/store_key.rs:35`).

`retrieve` is where the wallet boundary is enforced: it checks the owner pid, then refuses if the key is a
`Secp256k1Eth` key (`AccessDenied`), then refuses a locked key (`Locked`), and only otherwise returns a
clone of the bytes and bumps `use_count` (`store/retrieve.rs:22`). A wallet secret is therefore never
exportable through `retrieve`; the only way to use it is to ask the keyring to sign, which never returns
the secret. Non-wallet key types (symmetric material, HMAC secrets, and so on) are retrievable by their
owner.

`delete` verifies the owner pid, removes the entry, and volatile-wipes the removed key bytes before
returning (`store/delete.rs:29`). `lock`/`unlock` flip the `locked` flag owner-checked
(`store/lock.rs:20`, `store/unlock.rs:20`). `metadata` returns a fixed 36-byte record
`id(4) || key_type(1) || size(2) || owner_pid(4) || created_at(8) || expires_at(8) || use_count(8) ||
locked(1)` (`handlers/metadata.rs:41`); it exposes the counters but not the key. `count` returns the
number of keys owned by the caller (`store/count.rs:20`).

### Wallet operations

| Op | Code | Request payload | Reply body | Handler |
|---|---|---|---|---|
| `WALLET_IMPORT` | 8 | `pid(4) || now(8) || expires(8) || secret(32)` | `id(4)` | `handlers/wallet_import.rs:22` |
| `WALLET_GENERATE` | 9 | `pid(4) || now(8) || expires(8)` | `id(4)` | `handlers/wallet_generate.rs:22` |
| `WALLET_ADDRESS` | 10 | `pid(4) || id(4)` | `address(20)` | `handlers/wallet_address.rs:22` |

`wallet_generate` draws a 32-byte secret from `crypto_random` (the kernel secure RNG) and validates it as
a proper secp256k1 scalar with `eth_secret_valid`, retrying up to 32 times; on 32 failures it wipes the
scratch and returns `EINVAL` (`wallet_generate.rs:37`). `eth_secret_valid` rejects the all-zero scalar and
any value at or above the curve order `n` by a constant big-endian comparison against `N`
(`store/eth_valid.rs:17`). The secret is stored as `KeyType::Secp256k1Eth` and the local scratch is
volatile-zeroed whether the store succeeded or not (`wallet_generate.rs:50`).

`wallet_import` takes a caller-supplied 32-byte secret, validates it the same way, stores it, and wipes
the scratch (`wallet_import.rs:37`). This is the one op where a secret rides in on the request wire, which
is exactly why the server loop wipes the receive buffer after every reply (below).

`wallet_address` derives the Ethereum address without exporting the key: it takes the secret owner-checked,
computes the uncompressed secp256k1 public key, wipes the secret, then Keccak-256-hashes the 64-byte public
key and returns the low 20 bytes of the digest (`wallet_address.rs:33`, and the same derivation is factored
into `address_of`, `src/server/ethaddr.rs:17`).

### Signing operations

| Op | Code | Request payload | Reply body | Handler |
|---|---|---|---|---|
| `SIGN_NOX_RECEIPT` | 11 | `pid(4) || id(4) || capsule_id(32) || publisher(20) || amount(32) || nonce(32) || epoch(32) || expiry(32) || type(32)` | `user(20) || struct_hash(32) || sig(65)` | `handlers/sign_receipt/sign_receipt.rs:26` |
| `SIGN_NOX_APPROVE` | 12 | `pid(4) || id(4) || nonce(32) || maxPriority(32) || maxFee(32) || gas(32) || amount(32)` | raw signed tx | `handlers/sign_approve.rs:25` |
| `SIGN_ETH_TRANSFER` | 13 | `pid(4) || id(4) || to(20) || nonce(32) || maxPriority(32) || maxFee(32) || gas(32) || value(32)` | raw signed tx | `handlers/sign_eth_transfer.rs:25` |

All three share the same shape: resolve the caller, take the secret owner-checked through `eth_secret`
(which itself re-checks the owner pid, the `Secp256k1Eth` type, the lock flag, and the 32-byte length,
`store/eth_secret.rs:20`), build the message, hash it, sign it, and zero the secret the instant the
signature is out.

`sign_eth_transfer` is the fullest illustration. The request is exactly 188 bytes (`4 + 4 + 20 + 32*5`,
`sign_eth_transfer.rs:26`). It builds the unsigned EIP-1559 payload with the capsule's own RLP encoder,
Keccak-256-hashes it, signs the digest with secp256k1 producing a 65-byte `r || s || v` recoverable
signature, and re-encodes the signed transaction with recovery parity `v - 27`
(`sign_eth_transfer.rs:49`). The secret is zeroed both on the keccak-failure branch
(`sign_eth_transfer.rs:55`) and immediately after the sign call (`sign_eth_transfer.rs:60`), and the
handler rejects a signature whose `v` byte is below 27.

`sign_approve` signs a NOX ERC-20 `approve` as an EIP-1559 transaction against the settlement contract,
same retrieve-sign-wipe path over a 164-byte request (`4 + 4 + 32*5`, `sign_approve.rs:26`); the NOX token
and settlement addresses, the `approve` selector `0x095ea7b3`, and the chain id `1` are fixed constants
(`src/server/eip1559/consts.rs:17`).

`sign_receipt` is the EIP-712 payment path. It derives the signer's own address from the secret
(`address_of`), builds the typed `ReceiptFields` (`sign_receipt/read_fields.rs:19`), computes the struct
hash and the EIP-712 digest under a fixed type hash and domain separator
(`src/server/eip712/consts.rs:17`), signs, wipes the secret, and returns the signer address, the struct
hash, and the 65-byte signature (`sign_receipt/sign_receipt.rs:26`). This is the op the
[payment](payment.md) capsule calls (`KEYRING_OP_SIGN_RECEIPT = 11`,
`userland/capsule_payment/src/server/consts.rs:21`).

### Wallet rail listing

| Op | Code | Request payload | Reply body | Handler |
|---|---|---|---|---|
| `LIST_WALLET_RAILS` | 14 | none | count + rail records | `handlers/list_wallet_rails.rs:22` |

`list_wallet_rails` needs no caller pid and takes no key; it serializes a static table of four rails
(ETH, NOX, PR, SAL) with their family, status, flags, chain id, contract address, and symbol
(`src/server/wallet_rail/registry.rs:21`, encoded by `wallet_rail/encode.rs:21`). It is descriptive only:
it exposes no secret and touches no store entry.

## Architecture and lifecycle

The three top-level modules are `protocol` (the wire frame, errno set, and op constants), `server` (the
loop, the caller check, the handlers, the RLP and EIP-712 encoders, and the scratch-wiping helpers), and
`store` (the key store and its owner-checked operations) (`src/main.rs:22`).

The `Store` is a `BTreeMap<u32, KeyEntry>` keyed by an auto-incrementing id, with a `next_id`
(`src/store/types/store.rs:20`). A `KeyEntry` carries the material and its metadata
(`src/store/types/key_entry.rs:20`):

```
  struct KeyEntry {
      key_type: KeyType,     // Secp256k1Eth (8), Symmetric (0), SigningKey (7), ...
      data: Vec<u8>,         // the raw key material, 1..=256 bytes
      owner_pid: u32,        // the pid that stored it
      created_at: u64, expires_at: u64,
      use_count: u64,        // incremented on each retrieve / secret use
      locked: bool,          // retrieval- and signing-blocking flag
  }

  impl Drop for KeyEntry:  secure_wipe(self.data)     // zero on drop
```

The store is capped at 128 keys and each key at 256 bytes (`src/store/types/constants.rs:16`), so it
cannot grow without bound. The destructor is the load-bearing detail: when a `KeyEntry` is dropped,
`secure_wipe` volatile-zeroes its `data` with a compiler fence (`src/store/wipe.rs:19`), so key material
is erased from the heap when a key is deleted or the store is torn down rather than left in a freed
allocation.

Lifecycle:

1. The kernel spawns the capsule right after ramfs, in the core service fleet (`spawn_after_ramfs`,
   `src/userspace/init/spawn_plan/core.rs:23`). `spawn_keyring` calls `boot::capsule("KEYRING",
   "keyring", ...)` from the kernel embed at `src/security/keyring_capsule/`
   (`spawn_plan/core.rs:45`).
2. `capsule_boot::boot` verifies the embedded ELF, cert, manifest, and attestation, registers the
   `keyring` service on port 4098, and on success logs `[KEYRING] capsule spawned`; on failure it logs an
   `[ERROR]` line describing the `SpawnError` (`src/userspace/init/capsule_boot/run.rs:29`).
3. `_start` initializes the heap and enters `server::run`, which never returns
   (`src/main.rs:29`).
4. The server loop drives the whole capsule (`src/server/runner.rs:28`):

```
  run():
      buf   = vec![0u8; 4096]
      store = Store::new()                         // empty BTreeMap, next_id = 1
      loop:
          n = mk_ipc_recv_from(inbox 0, buf, &sender_pid)
          if n <= 0: continue
          match decode_request(buf[..n]):
              Some(req) => resp = dispatch(store, req, sender_pid)
              None      => wipe(buf[..n]); continue      // undersized frame
          mk_ipc_send(KERNEL_REPLY_ENDPOINT, resp)
          wipe(buf[..n])                                 // volatile-zero after every reply
```

The buffer is wiped both when decode fails (`runner.rs:41`) and after every successful reply
(`runner.rs:46`), so a request that carried key material (a `wallet_import`, for instance) does not linger
in the receive buffer once it is handled.

## Protocol and IPC

The keyring is a pure server: it registers the `keyring` service on port 4098 and answers, but makes no
outbound calls of its own. Its callers open the service by name through the registry and issue a
synchronous `mk_ipc_call`, framing `seq(4) || op(2) || pad(2) || payload` and reading back
`seq(4) || status(4) || body`.

- [login](login.md) toggles the lock. Its keyring client sends `OP_LOCK = 4` and `OP_UNLOCK = 5` with a
  `caller_pid(4) || key_id(4)` payload and reads the 4-byte status (`userland/capsule_login/src/clients/keyring.rs:8`).
  Login unlocks the wallet key on a successful sign-in and locks it again when the session ends, which is
  how the lock flag becomes an authenticated-session gate on signing.
- The [wallet](wallet-nonos.md) drives the wallet ops: `OP_WALLET_GENERATE = 9`, `OP_WALLET_ADDRESS = 10`,
  `OP_SIGN_NOX_APPROVE = 12`, `OP_SIGN_ETH_TRANSFER = 13`, and `OP_LIST_WALLET_RAILS = 14`
  (`userland/capsule_wallet_nonos/src/wallet/ipc/constants.rs:19`), through a shared `keyring_call`
  helper that frames the request and checks the status (`.../wallet/ipc/call.rs:23`). The wallet never
  sees a secret: it asks for an address or a signed transaction and renders the reply.
- The [payment](payment.md) capsule calls `OP_SIGN_NOX_RECEIPT = 11` to settle a paid request
  (`KEYRING_OP_SIGN_RECEIPT`, `userland/capsule_payment/src/server/consts.rs:21`); it discovers the
  keyring port through the registry and passes the owner pid and wallet key id
  (`.../capsule_payment/src/server/handlers/pay.rs:43`).

The reply always leaves through `KERNEL_REPLY_ENDPOINT = 0x1_0000_0002` (`src/protocol/types.rs:32`), the
endpoint id named in the manifest's reply record; the kernel routes that frame back to the caller.

## Security analysis

- **The keyring is the sole key holder.** Applications, including the wallet and the payment service,
  request signatures and receive signed transactions or receipts; they never receive a private key. A
  compromise of the wallet or payment capsule exposes its UI and IPC path but not a signing key.
- **Wallet keys are non-exportable, structurally.** A `Secp256k1Eth` key cannot leave the capsule through
  `retrieve` (`store/retrieve.rs:27`), and the signing and address handlers return signatures, hashes, and
  the 20-byte address, never the 32 secret bytes. There is no op that emits a wallet secret.
- **Owner-pid isolation on every operation**, backed by [caller attestation](#caller-attestation), means
  one capsule cannot use, read, lock, delete, or even read metadata for another capsule's key. The owner
  check is enforced twice for signing: once when the caller is resolved and again inside `eth_secret`
  (`store/eth_secret.rs:22`).
- **The lock is a gate, not a second cryptographic factor.** `lock` sets a flag that blocks both
  `retrieve` (`EBUSY`) and `eth_secret` (which is what every signer uses), so a locked wallet key cannot
  be signed with; login uses this to bind signing to an authenticated session. But the lock does not
  re-encrypt or cryptographically bind the key, it only refuses the operation while set, so it is an
  access gate rather than at-rest protection.
- **Generation quality.** Generated keys come from the kernel secure RNG and are validated as proper
  secp256k1 scalars in `[1, n-1]` with rejection sampling (`store/eth_valid.rs`, `wallet_generate.rs:37`).

The mask is the least-privilege argument that matters most in the whole tree, because this capsule holds
every private key. It is `0x39` = IPC + Memory + Crypto (`Capsule.mk:17`). It holds Crypto because it
signs and generates keys through the kernel crypto and RNG syscalls, and IPC and Memory to be a service
with a heap. It holds *nothing else*, and the absence is the point: no Driver, Mmio, Irq, Dma, or Pio, so
the capsule that holds every private key cannot touch a device or program a DMA engine that could
exfiltrate those keys over hardware; no FileSystem, so it cannot write a key to a storage surface (keys
are RAM-only by construction, not discipline alone); no Network, so it cannot open a socket and ship a
secret off the machine; no Debug, so it cannot open a serial surface or emit a diagnostic line that could
leak key bytes to a log. Its isolation from the wallet is the whole design: the wallet reaches the keyring
only over IPC, so compromising the wallet exposes its UI and network path but not a key. The honest bound
is that the mask cannot forbid a *logic* leak, a handler that returned key bytes in a reply body would
defeat it, which is why the signers return signatures and the retrieval path hard-refuses the wallet key
type.

### The wiping discipline

The keyring wipes key material at four distinct points, and it is worth listing them exactly because
together they are the capsule's memory posture:

```
  1. KeyEntry::drop              secure_wipe(data)      key erased on delete and on store teardown
  2. store::delete               secure_wipe(data)      the removed entry's bytes wiped before return
  3. the wallet handlers         zeroize32 / volatile   the signing/derivation scratch wiped after use
  4. the server loop             wipe(buf)              the request buffer wiped after every reply
```

Each is a volatile write plus a compiler fence, so the optimizer cannot elide it: `secure_wipe`
(`src/store/wipe.rs:19`), `zeroize32` (`src/server/zeroize.rs:17`), and the loop `wipe`
(`src/server/wipe.rs:19`). The signing scratch is zeroed the moment the signature is produced, and on the
error branches too: `sign_eth_transfer` zeroes on the keccak failure and after the sign
(`sign_eth_transfer.rs:55`, `:60`), and `sign_approve`, `sign_receipt`, and `wallet_address` do the same
(`sign_approve.rs:52`,`:57`; `sign_receipt.rs:46`,`:61`; `wallet_address.rs:40`). `wallet_generate` and
`wallet_import` volatile-zero their 32-byte scratch on both the success and the reject paths
(`wallet_generate.rs:44`,`:50`; `wallet_import.rs:38`,`:44`).

Two honest notes on the discipline. First, note 2 (`store::delete`) overlaps note 1 (the `Drop` on the
same `KeyEntry` that `delete` removes and then drops at end of scope), so the delete path wipes the bytes
twice; that is belt-and-suspenders, not a gap. Second, `eth_secret` and `retrieve` hand back an owned copy
of the key (`store/eth_secret.rs:34`, a `[u8; 32]`; `store/retrieve.rs:34`, a `Vec` clone). The signing
handlers wipe their `[u8; 32]` copy; the plain `retrieve` reply for a non-wallet key type is copied into
the response `Vec` and then the receive buffer is wiped, but the response `Vec` itself is not explicitly
zeroed after `mk_ipc_send`. In practice a wallet secret never travels that path because `retrieve` refuses
the `Secp256k1Eth` type outright, so the un-zeroed reply buffer only ever carries non-wallet material the
caller asked to read back.

The net result: a private key exists in cleartext only inside the store's `KeyEntry.data` (wiped on drop
and on delete) and, momentarily, in a signing or derivation scratch buffer (wiped after use on every
branch). It is never left in a freed allocation or a stale request buffer.

### Honest gaps

Stated plainly: `expires_at` is stored and reported in metadata but never enforced, so a key does not
expire on its own; the lock is an access gate, not at-rest encryption or a second factor; all signing is
single-signer, with no threshold or multi-party path; `use_count` is tracked but not surfaced as an audit
trail; `LIST_WALLET_RAILS` returns a static table; and keys are held in RAM only, consistent with the
RAM-resident posture, so there is no persistent, at-rest-encrypted key store here. The `PR` and `SAL`
rails in the rail table are marked config-required and reserved (`wallet_rail/registry.rs:38`), not live.

## How to contribute

The source lives at `userland/capsule_keyring/`. The wire protocol is under `src/protocol/`, the server
loop and handlers under `src/server/`, and the key store under `src/store/`.

To add a new operation:

1. Add the opcode constant to `src/protocol/types.rs:17` (keep the numbering contiguous; the wire numbers
   are load-bearing for callers).
2. Write the handler as one file per op under `src/server/handlers/`, exposing a
   `pub fn op(store: &mut Store, req: Request<'_>, sender_pid: u32) -> Vec<u8>` that returns an encoded
   response. Length-check the payload first, then call `resolve_caller` before touching any key, and scope
   every store access to the resolved caller pid. Return errors with `encode_response(req.seq, ERRNO,
   &[])`, never a panic.
3. Re-export the handler from `src/server/handlers/mod.rs:32` and wire it into the match in
   `src/server/dispatch.rs:28`.
4. If the op touches the store, add the owner-checked method under `src/store/` (one file per operation,
   the way `store/lock.rs` and `store/eth_secret.rs` are split) and re-export it through
   `src/store/mod.rs`.
5. If the op handles a secret, wipe every transient copy with `zeroize32` or a volatile loop on every
   return path, including the error branches, matching the existing signers.

To build and sign the capsule, use the per-slug make targets generated by the shared capsule rules
(`nonos-mk/capsule.mk:182`, `:261`, `:263`, `:184`, included through `userland/capsule_keyring/Capsule.mk:20`):

```
  make nonos-mk-keyring               build the capsule ELF
  make nonos-mk-keyring-sign          produce the id cert, manifest, and attestation trailer
  make nonos-mk-keyring-verify        verify the signed artifacts against the trust anchor
  make nonos-mk-check-keyring-keys    check the per-capsule signing keys exist
```

For a bootable image and a round-trip test, `make nonos-mk-keyring-prod` builds a kernel profile with the
`microkernel-keyring` feature (`Makefile:910`), and `make nonos-mk-boot-keyring` runs the boot round-trip
harness `tests/boot/keyring_round_trip.sh` (`Makefile:1384`), which is part of `make nonos-mk-test`
(`Makefile:1478`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns errors as a negative status, never a panic); modular
files, one unit per file, with `mod.rs` used only for re-exports; and the AGPL header at the top of every
source file, matching the header on every existing module.

## Debugging

Because the keyring deliberately holds no Debug cap, it emits no diagnostic output of its own; debugging is
done from the boot marker and the caller side.

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[KEYRING] capsule spawned` from the boot log (tag `KEYRING`, message `capsule spawned`,
`src/userspace/init/capsule_boot/run.rs:29`, format in `src/sys/boot_log/output.rs:33`). An absent marker
means the service never registered, so `mk_service_lookup("keyring")` will not resolve for the wallet, the
payment capsule, or login, and every wallet operation and paid receipt fails at the caller; the error path
prints an `[ERROR]` line describing the `SpawnError` instead (usually a signature, manifest, or capability
failure).

Request-time failures are read from the reply status:

- `EACCES` (-13) is `resolve_caller` refusing a payload pid that did not match the attested sender, or the
  owner-pid check on a key that belongs to another capsule. A wallet that can show an address but cannot
  sign is often this boundary doing its job against a caller acting on a key it does not own.
- `EBUSY` (-16) is a retrieve or a sign against a locked key. If signing fails right after boot or after a
  session ends, the wallet key is locked; login unlocks it on sign-in (`OP_UNLOCK = 5`).
- `ENOENT` (-2) is a key id that does not exist in the store.
- `EINVAL` (-22) from `wallet_generate` means rejection sampling failed to draw a valid secp256k1 scalar
  in 32 tries (a degraded RNG); from a signer it means a bad request length or a failed keccak/sign step;
  from `store` it means an empty or over-256-byte body.
- `ENOSPC` (-28) means the 128-key store is full.

Since the keyring never logs, the caller's own markers are the trace: the wallet and payment capsules log
their side of each call, and a mismatch between "keyring absent at boot" and "wallet cannot sign" points
squarely at the spawn, not the request.

## Source map

```
  src/main.rs                              _start -> heap_init -> server::run
  src/protocol/types.rs                    op constants, HDR_LEN, KERNEL_REPLY_ENDPOINT, Request
  src/protocol/{decode,encode}.rs          request decode and response encode
  src/protocol/errno.rs                    ENOENT EACCES EBUSY EINVAL ENOSPC
  src/server/runner.rs                     the loop + buffer wipe
  src/server/dispatch.rs                   op -> handler dispatch
  src/server/caller.rs                     resolve_caller (no-impersonation rule)
  src/server/handlers/                     one file per op (store/retrieve/delete/lock/unlock/...)
  src/server/handlers/sign_eth_transfer.rs, sign_approve.rs, sign_receipt/   the signers
  src/server/handlers/wallet_generate.rs, wallet_import.rs, wallet_address.rs   wallet creation and address
  src/server/eip1559/                      the EIP-1559 RLP encoders + NOX approve constants
  src/server/eip712/                       the EIP-712 receipt type hash, domain, and digest
  src/server/ethaddr.rs                    address_of (pubkey -> keccak -> 20 bytes)
  src/server/wallet_rail/                  the static rail table and its encoder
  src/server/{wipe,zeroize}.rs             the buffer and scratch wipes
  src/store/types/key_entry.rs             KeyEntry + secure-wipe Drop
  src/store/types/{store,key_type,store_error,key_metadata,constants}.rs   the store model
  src/store/{store_key,retrieve,delete,lock,unlock,metadata,count,eth_secret,eth_valid}.rs   store ops
  src/store/wipe.rs                        secure_wipe
  Capsule.mk                               slug, handle, ports, capability mask, kernel mirror
  src/security/keyring_capsule             the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/core.rs    the core-fleet spawn entry
  userland/capsule_login/src/clients/keyring.rs        the login lock/unlock client
  userland/capsule_wallet_nonos/src/wallet/ipc/        the wallet keyring client
  userland/capsule_payment/src/server/consts.rs        the payment receipt-sign op
  nonos-mk/capsule.mk                      the generated nonos-mk-keyring[-sign|-verify] targets
```

Every reference above is verified against those trees.

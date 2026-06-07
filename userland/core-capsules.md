# Core Service Capsules

This page documents the core non-GUI service capsules: RAMFS, VFS, keyring,
entropy, crypto, policy, attest, market, installer, payment, power, and
`proof_io`. Read [Services](services.md), [Protocol Atlas](protocols.md), and
[Runtime Workflows](workflows.md) first.

Use this page as a per-capsule audit map. Each service is described by the
state it owns, the request router it exposes, and the failure boundary a caller
will see.

---

## 1. Common Service Shape

Most core capsules receive IPC, parse a protocol request, dispatch by operation,
mutate capsule-owned state when needed, and reply with a protocol status. The
shape is visible in RAMFS, VFS, keyring, entropy, crypto, installer, and payment
dispatchers (`userland/capsule_ramfs/src/server/dispatch.rs:26`,
`userland/capsule_vfs/src/server/dispatch.rs:26`,
`userland/capsule_keyring/src/server/dispatch.rs:27`,
`userland/capsule_entropy/src/server/dispatch.rs:25`,
`userland/capsule_crypto/src/server/dispatch.rs:27`,
`userland/capsule_installer/src/server/dispatch.rs:22`,
`userland/capsule_payment/src/server/dispatch.rs:25`).

```
+--------------------------+
| service inbox            |
+------------+-------------+
             |
+------------+-------------+
| protocol parse           |
+------------+-------------+
             |
+------------+-------------+
| op dispatch              |
+------------+-------------+
             |
+------------+-------------+
| state read or mutation   |
+------------+-------------+
             |
+------------+-------------+
| encoded response         |
+--------------------------+
```

## 2. Storage and Filesystem Services

RAMFS owns encrypted file records in memory. Each file stores a per-file key,
nonce, and ciphertext buffer, and the store maps names to file records
(`userland/capsule_ramfs/src/store/types.rs:23`,
`userland/capsule_ramfs/src/store/types.rs:24`,
`userland/capsule_ramfs/src/store/types.rs:25`,
`userland/capsule_ramfs/src/store/types.rs:26`,
`userland/capsule_ramfs/src/store/types.rs:29`,
`userland/capsule_ramfs/src/store/types.rs:30`). Its dispatcher handles open,
read, write, truncate, and close, and rejects unknown operations with `EINVAL`
(`userland/capsule_ramfs/src/server/dispatch.rs:27`,
`userland/capsule_ramfs/src/server/dispatch.rs:28`,
`userland/capsule_ramfs/src/server/dispatch.rs:29`,
`userland/capsule_ramfs/src/server/dispatch.rs:30`,
`userland/capsule_ramfs/src/server/dispatch.rs:31`,
`userland/capsule_ramfs/src/server/dispatch.rs:32`,
`userland/capsule_ramfs/src/server/dispatch.rs:33`).

VFS owns file entries and open file descriptors. Its store caps file count, open
FD count, and file size, and each open FD records file index, owner pid,
position, append mode, and writable state
(`userland/capsule_vfs/src/store/fdtable/types.rs:20`,
`userland/capsule_vfs/src/store/fdtable/types.rs:21`,
`userland/capsule_vfs/src/store/fdtable/types.rs:22`,
`userland/capsule_vfs/src/store/fdtable/types.rs:37`,
`userland/capsule_vfs/src/store/fdtable/types.rs:43`,
`userland/capsule_vfs/src/store/fdtable/types.rs:44`,
`userland/capsule_vfs/src/store/fdtable/types.rs:45`,
`userland/capsule_vfs/src/store/fdtable/types.rs:46`,
`userland/capsule_vfs/src/store/fdtable/types.rs:47`,
`userland/capsule_vfs/src/store/fdtable/types.rs:48`). Its dispatcher covers
open, close, read, write, stat, list, mkdir, unlink, rename, and healthcheck
(`userland/capsule_vfs/src/server/dispatch.rs:27` to
`userland/capsule_vfs/src/server/dispatch.rs:38`).

```
+--------------------------+
| caller path request      |
+------------+-------------+
             |
+------------+-------------+
| ramfs encrypted files    |
| vfs files and fd table   |
+------------+-------------+
             |
+------------+-------------+
| handler validates owner  |
| handler validates bounds |
+------------+-------------+
             |
+------------+-------------+
| data or errno reply      |
+--------------------------+
```

## 3. Security and Crypto Services

Keyring owns a `BTreeMap` from key id to key entry and a `next_id` counter
(`userland/capsule_keyring/src/store/types/store.rs:20`,
`userland/capsule_keyring/src/store/types/store.rs:21`,
`userland/capsule_keyring/src/store/types/store.rs:22`). Its dispatch surface
covers key store, retrieve, delete, lock, unlock, metadata, count, wallet import,
wallet generate, wallet address, NOX receipt signing, and NOX approve signing
(`userland/capsule_keyring/src/server/dispatch.rs:28` to
`userland/capsule_keyring/src/server/dispatch.rs:41`).

Entropy owns atomic counters for request count, bytes served, last reseed
request, and source failures (`userland/capsule_entropy/src/pool/types.rs:26`,
`userland/capsule_entropy/src/pool/types.rs:27`,
`userland/capsule_entropy/src/pool/types.rs:28`,
`userland/capsule_entropy/src/pool/types.rs:29`,
`userland/capsule_entropy/src/pool/types.rs:30`). Its dispatcher covers random
bytes, stats, reseed, and healthcheck
(`userland/capsule_entropy/src/server/dispatch.rs:26` to
`userland/capsule_entropy/src/server/dispatch.rs:31`).

Crypto is a stateless request dispatcher for hash, signature, AEAD, X25519,
HMAC, HKDF, and healthcheck operations. The routing table maps each op directly
to its handler and rejects unknown operations with `EINVAL`
(`userland/capsule_crypto/src/server/dispatch.rs:28` to
`userland/capsule_crypto/src/server/dispatch.rs:43`).

Policy uses a fixed header. The runner polls its endpoint, validates header
length, decodes the header, checks payload length, decodes the field, then
dispatches `OP_GET` or `OP_SET`
(`userland/capsule_policy/src/server/runner.rs:23`,
`userland/capsule_policy/src/server/runner.rs:27`,
`userland/capsule_policy/src/server/runner.rs:32`,
`userland/capsule_policy/src/server/runner.rs:36`,
`userland/capsule_policy/src/server/runner.rs:40`,
`userland/capsule_policy/src/server/runner.rs:45`,
`userland/capsule_policy/src/server/runner.rs:52`,
`userland/capsule_policy/src/server/runner.rs:53`,
`userland/capsule_policy/src/server/runner.rs:54`).

Attest parses a request and routes healthcheck, proof summary, proof invariants,
proof boot, and proof capsule list. Unknown ops return `E_BAD_OP`
(`userland/capsule_attest/src/server/handlers/router.rs:24`,
`userland/capsule_attest/src/server/handlers/router.rs:29`,
`userland/capsule_attest/src/server/handlers/router.rs:30`,
`userland/capsule_attest/src/server/handlers/router.rs:31`,
`userland/capsule_attest/src/server/handlers/router.rs:32`,
`userland/capsule_attest/src/server/handlers/router.rs:33`,
`userland/capsule_attest/src/server/handlers/router.rs:34`,
`userland/capsule_attest/src/server/handlers/router.rs:35`).

```
+--------------------------+
| keyring wallet request   |
| entropy random request   |
| crypto primitive request |
| policy field request     |
| attest proof request     |
+------------+-------------+
             |
+------------+-------------+
| protocol-specific parse  |
+------------+-------------+
             |
+------------+-------------+
| handler or errno         |
+--------------------------+
```

## 4. Market, Install, Payment, Power, and Proof

Market allocates receive and transmit buffers, decodes a market request, checks
body bounds, then dispatches healthcheck, load index, list apps, get app, get
release, and install-ready operations (`userland/capsule_market/src/server/runner.rs:32`,
`userland/capsule_market/src/server/runner.rs:33`,
`userland/capsule_market/src/server/runner.rs:34`,
`userland/capsule_market/src/server/runner.rs:41`,
`userland/capsule_market/src/server/runner.rs:49`,
`userland/capsule_market/src/server/runner.rs:57`,
`userland/capsule_market/src/server/runner.rs:58`,
`userland/capsule_market/src/server/runner.rs:59`,
`userland/capsule_market/src/server/runner.rs:60`,
`userland/capsule_market/src/server/runner.rs:61`,
`userland/capsule_market/src/server/runner.rs:62`,
`userland/capsule_market/src/server/runner.rs:63`).

Installer dispatches healthcheck and install. Payment dispatches healthcheck,
pay, and drain receipts (`userland/capsule_installer/src/server/dispatch.rs:23`,
`userland/capsule_installer/src/server/dispatch.rs:24`,
`userland/capsule_installer/src/server/dispatch.rs:25`,
`userland/capsule_payment/src/server/dispatch.rs:26`,
`userland/capsule_payment/src/server/dispatch.rs:27`,
`userland/capsule_payment/src/server/dispatch.rs:28`,
`userland/capsule_payment/src/server/dispatch.rs:29`).

Power parses a request, then routes healthcheck, reboot, and shutdown, with
unknown ops returned as `E_BAD_OP`
(`userland/capsule_power/src/server/handlers/router.rs:22`,
`userland/capsule_power/src/server/handlers/router.rs:23`,
`userland/capsule_power/src/server/handlers/router.rs:27`,
`userland/capsule_power/src/server/handlers/router.rs:28`,
`userland/capsule_power/src/server/handlers/router.rs:29`,
`userland/capsule_power/src/server/handlers/router.rs:30`,
`userland/capsule_power/src/server/handlers/router.rs:31`).

`proof_io` is not a long-lived IPC service. Its `_start` checks time calls,
unknown syscall number handling, invalid debug pointer handling, invalid debug
size handling, retired syscall rejection, then emits pass or fail and exits
(`userland/capsule_proof_io/src/main.rs:37`,
`userland/capsule_proof_io/src/main.rs:38`,
`userland/capsule_proof_io/src/main.rs:44`,
`userland/capsule_proof_io/src/main.rs:48`,
`userland/capsule_proof_io/src/main.rs:52`,
`userland/capsule_proof_io/src/main.rs:56`,
`userland/capsule_proof_io/src/main.rs:64`,
`userland/capsule_proof_io/src/main.rs:65`).

| Capsule | Audit question | First source |
|---------|----------------|--------------|
| `market` | Was the index decoded, bounded, and dispatched to the right handler? | `userland/capsule_market/src/server/runner.rs:41` |
| `installer` | Is the request healthcheck or install? | `userland/capsule_installer/src/server/dispatch.rs:23` |
| `payment` | Is the request pay or drain receipts? | `userland/capsule_payment/src/server/dispatch.rs:26` |
| `power` | Is the command reboot or shutdown after parse? | `userland/capsule_power/src/server/handlers/router.rs:27` |
| `proof_io` | Which syscall proof failed before exit? | `userland/capsule_proof_io/src/main.rs:38` |

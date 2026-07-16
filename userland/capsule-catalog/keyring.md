# capsule_keyring

`capsule_keyring` is the system's secure key store and the only capsule that holds private keys. It
stores key material behind an owner-pid boundary and a lock state, generates and imports wallets, and
signs Ethereum and NOX transactions, and it is scrupulous about memory: secrets are wiped when an entry
is dropped, wiped after every signing use, wiped after generation, and the request buffer is wiped after
every reply. No other capsule, including the [wallet](wallet-nonos.md), ever holds a private key. Service
`keyring` on port 4098, reply endpoint `0x1_0000_0002`, capability mask `0x39`. The source is
`userland/capsule_keyring/`.

## Contents

- [The server loop](#the-server-loop)
- [The wire protocol](#the-wire-protocol)
- [Caller attestation](#caller-attestation)
- [The store model](#the-store-model)
- [Store, retrieve, lock](#store-retrieve-lock)
- [Wallet generation](#wallet-generation)
- [Ethereum signing in full](#ethereum-signing-in-full)
- [The wiping discipline](#the-wiping-discipline)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop

`main.rs:30` initializes the heap and calls `server::run` (`src/server/runner.rs:28`), which drives the
loop over a 4 KiB buffer and wipes that buffer around every request:

```
  run():
      store = Store::new()                        // BTreeMap<u32, KeyEntry>, next_id = 1
      loop:
          n = mk_ipc_recv_from(inbox 0, buf, &sender_pid)
          req = decode_request(buf[..n])          // else wipe(buf[..n]); continue
          resp = dispatch(store, req, sender_pid)
          mk_ipc_send(KERNEL_REPLY_ENDPOINT, resp)
          wipe(buf[..n])                           // volatile-zero the buffer after reply
```

The buffer is wiped both when a decode fails (`runner.rs:41`) and after every successful reply
(`runner.rs:46`), so a request that carried key material (a wallet import, for instance) does not linger
in the receive buffer once it has been handled.

## The wire protocol

The frame is an 8-byte header (`src/protocol/types.rs:34`), a `u32` sequence and a `u16` op, followed by
a payload; there is no magic. Fourteen operations (`src/protocol/types.rs:17`):

```
  1  STORE            6  METADATA        11  SIGN_NOX_RECEIPT
  2  RETRIEVE         7  COUNT           12  SIGN_NOX_APPROVE
  3  DELETE           8  WALLET_IMPORT   13  SIGN_ETH_TRANSFER
  4  LOCK             9  WALLET_GENERATE 14  LIST_WALLET_RAILS
  5  UNLOCK          10  WALLET_ADDRESS
```

Every payload begins with a four-byte caller pid, and the response is the sequence number, a status, and
an optional body (`encode_response`).

## Caller attestation

`resolve_caller` (`src/server/caller.rs`) is the same no-impersonation rule as the [vfs pool](vfs.md):

```
  resolve_caller(payload_pid, sender_pid):
      if sender_pid == 0:            payload_pid    // kernel-side TCB, trusted
      if payload_pid == sender_pid:  sender_pid     // ring-3 caller must match its attested pid
      else:                          None -> EACCES
```

A ring-3 capsule can only act on its own keys, because its claimed pid must equal the kernel-attested
`sender_pid`, and every key operation is scoped to the caller pid, so no capsule can retrieve, sign with,
or delete another capsule's key.

## The store model

The `Store` (`src/store/types/store.rs:20`) is a `BTreeMap<u32, KeyEntry>` keyed by an auto-incrementing
id, and a `KeyEntry` (`src/store/types/key_entry.rs:20`) carries the material and its metadata:

```
  struct KeyEntry {
      key_type: KeyType,     // Secp256k1Eth, ...
      data: Vec<u8>,         // the raw key material, 1..=256 bytes
      owner_pid: u32,        // the pid that stored it
      created_at: u64, expires_at: u64,
      use_count: u64,        // incremented per use
      locked: bool,          // retrieval-blocking flag
  }

  impl Drop for KeyEntry:  secure_wipe(self.data)     // <-- zero on drop
```

The store is capped at 128 keys and each key at 256 bytes, so it cannot be grown without bound. The
critical detail is the destructor: when a `KeyEntry` is dropped, `secure_wipe`
(`src/store/wipe.rs`) volatile-zeroes its `data` with a fence, so key material is erased from the heap
when a key is deleted or the store is torn down rather than left in a freed allocation.

## Store, retrieve, lock

`store` inserts a new `KeyEntry` under the next id with the attested owner pid; `retrieve` looks a key up
by id, checks `owner_pid` matches the caller (`AccessDenied` otherwise), and refuses a locked key with
`EBUSY`. `lock` and `unlock` toggle the flag, again owner-checked. The lock is advisory in that it blocks
retrieval but does not re-encrypt or cryptographically bind the key; it is a guard against accidental use,
not a second factor.

## Wallet generation

`wallet_generate` (`src/server/handlers/wallet_generate.rs:22`) creates a fresh Ethereum key and is
careful about both randomness quality and cleanup:

```
  wallet_generate(store, req, sender_pid):
      caller_pid = resolve_caller(payload_pid, sender_pid)
      (now, expires_at) = payload
      for _ in 0..32:                                  // rejection sampling
          crypto_random(secret, 32)
          if eth_secret_valid(secret):  break          // a valid secp256k1 scalar in [1, n-1]
      if not valid after 32 tries:  wipe secret; return EINVAL
      id = store.store(Secp256k1Eth, secret, caller_pid, now, expires_at)
      wipe secret                                       // erase the local copy either way
      return id
```

The secret is drawn from `crypto_random` (the kernel secure RNG) and validated as a proper secp256k1
scalar (`eth_secret_valid` rejects zero and out-of-range values), retrying up to 32 times, and the local
secret buffer is volatile-zeroed after it is copied into the store, so the transient copy does not linger.

## Ethereum signing in full

`sign_eth_transfer` (`src/server/handlers/sign_eth_transfer.rs:25`) is the full EIP-1559 signing path and
the clearest illustration of the keyring's discipline. The request is exactly 188 bytes,
`payload_pid(4) || id(4) || to(20) || 5 x 32-byte fields (nonce, maxPriorityFee, maxFee, gas, value)`:

```
  sign_eth_transfer(store, req, sender_pid):
      caller_pid = resolve_caller(payload_pid, sender_pid)         // else EACCES
      secret = store.eth_secret(id, caller_pid)                    // owner-checked; AccessDenied -> EACCES
      unsigned = unsigned_eth_transfer_payload(nonce, maxPriority, maxFee, gas, to, value)   // RLP
      digest   = keccak256(unsigned)                               // else zeroize(secret); EINVAL
      sig[65]  = secp256k1_sign(secret, digest)                    // r || s || v
      zeroize32(secret)                                            // <-- wipe immediately after signing
      require rc == 65 and sig[64] >= 27
      (r, s, v) = (sig[0..32], sig[32..64], sig[64] - 27)
      raw = signed_eth_transfer_tx((nonce, maxPriority, maxFee, gas, to, value), v, r, s)   // RLP
      return raw
```

The keyring builds the unsigned EIP-1559 transaction with its own RLP encoder
(`unsigned_eth_transfer_payload`), hashes it with Keccak-256, signs the digest with secp256k1 producing a
65-byte `(r, s, v)` recoverable signature, and re-encodes the signed transaction
(`signed_eth_transfer_tx`) with the recovery parity `v - 27`. The secret is retrieved owner-checked and
zeroed the instant the signature is produced (and also on the keccak-failure branch, `line 55`), so it is
live for the minimum window. The NOX signing ops (`SIGN_NOX_RECEIPT`, `SIGN_NOX_APPROVE`) follow the same
retrieve-sign-wipe shape for the NOX rails.

## The wiping discipline

Four separate wipes protect key material, and it is worth listing them because together they are the
keyring's security posture:

```
  1. KeyEntry::drop            secure_wipe(data)      key erased when deleted / store torn down
  2. sign_eth_transfer         zeroize32(secret)      signing secret erased right after the sign
  3. wallet_generate           volatile-zero secret   the transient generated secret erased after store
  4. the server loop           wipe(buf)              the request buffer erased after every reply
```

Each is a volatile write plus a compiler fence (`src/server/zeroize.rs:17`, `src/store/wipe.rs`) so the
optimizer cannot elide the erase. A private key therefore exists in cleartext only inside the store's
`KeyEntry.data` (wiped on drop) and, momentarily, in the signing scratch buffer (wiped after the sign);
it is never left in a freed allocation or a stale request buffer.

## Security analysis

- **The keyring is the sole key holder.** Applications, including the wallet, request signatures and
  receive signed transactions; they never receive a private key. A compromise of the wallet capsule
  exposes its UI and network path but not its signing key.
- **Owner-pid isolation** on every operation, backed by [caller attestation](#caller-attestation), means
  one capsule cannot use another's keys.
- **The wiping discipline** minimizes the lifetime of cleartext key material in memory.
- **Generation quality**: keys come from the kernel secure RNG and are validated as proper scalars.

The capability mask is `0x39` (`Capsule.mk`, `CAPSULE_REQUIRED_CAPS`), which decodes to CoreExec (1), IPC
(8), Memory (16), and Crypto (32). This is the least-privilege argument that matters most in the whole
tree, because the keyring is the sole private-key holder. It holds Crypto because it signs and generates
keys through the kernel crypto and RNG syscalls; it holds IPC and Memory to be a service with a heap. It
holds *nothing else*, and the absence is the security property: no Driver, Mmio, Irq, Dma, or Pio, so the
capsule that holds every private key cannot touch a device or program a DMA engine that could exfiltrate
those keys over hardware; no FileSystem, so it cannot write a key to a storage surface (keys are RAM-only
by construction, not by discipline alone); no Network, so it cannot open a socket and ship a secret off the
machine; no Debug, so it cannot open a serial surface or emit `MkDebug` that could leak key bytes to a log.
Its isolation from the [wallet](wallet-nonos.md) is the whole design: the wallet holds none of these
either, and reaches the keyring only over IPC, so compromising the wallet exposes its UI and network path
but not a key. The honest boundary is that the mask cannot forbid a *logic* leak, a signing handler that
returned key bytes in a reply body would defeat it, which is why the signers return signatures and hashes,
never the secret, and wipe the scratch immediately.

## Debugging

The service is `keyring` on port 4098 (`Capsule.mk`, `service:4098:keyring`), reply endpoint
`0x1_0000_0002`, and it is brought up in the boot fleet at `spawn_plan/core.rs:48` as
`boot::capsule("KEYRING", "keyring", ...)` from the kernel embed `src/security/keyring_capsule/`. Through
`capsule_boot::boot` that prints `[KEYRING] capsule spawned` on the boot log (framebuffer under
`NONOS_FBCONSOLE=1`), or a `[ERROR]` line with the `SpawnError`. A present marker means the service
registered under its manifest endpoint and `mk_service_lookup("keyring")` resolves for the wallet and the
[payment](payment.md) capsule; an absent marker means every wallet operation and every paid receipt will
fail because their signer never started. Because this capsule deliberately holds no Debug cap, it emits no
diagnostic output of its own by design (the NO LOGS invariant on the [attest](attest.md) page), so
debugging it is done from the boot marker and the caller side. The request-time failure signatures are the
tell: `EACCES` is `resolve_caller` refusing a payload pid that did not match the attested sender, or the
owner-pid check on a key that belongs to another capsule; `EBUSY` is a retrieve against a locked key;
`EINVAL` from `wallet_generate` means rejection sampling failed to draw a valid secp256k1 scalar in 32
tries (a degraded RNG). A wallet that can display an address but cannot sign is the owner-pid boundary
doing its job against a caller acting on a key it does not own.

## Honest gaps

Stated plainly: the `expires_at` timestamp is stored but not enforced, so a key does not expire on its
own; the lock is advisory (it blocks retrieval but does not cryptographically bind); all signing is
single-signer (no threshold or multi-party signing); `use_count` is tracked but not exposed as an audit
trail; and `LIST_WALLET_RAILS` returns a static list. The capsule holds keys in RAM only, consistent with
the RAM-resident posture, so there is no persistent, at-rest-encrypted key store here.

## Source map

```
  userland/capsule_keyring/src/server/runner.rs           the loop + buffer wipe
  userland/capsule_keyring/src/server/caller.rs            resolve_caller
  userland/capsule_keyring/src/server/handlers/store.rs, retrieve.rs, lock.rs    store ops
  userland/capsule_keyring/src/server/handlers/wallet_generate.rs, wallet_import.rs   wallet creation
  userland/capsule_keyring/src/server/handlers/sign_eth_transfer.rs, sign_nox_*.rs   the signers
  userland/capsule_keyring/src/server/eip1559.rs           the RLP encoders (unsigned/signed tx)
  userland/capsule_keyring/src/server/zeroize.rs           zeroize32
  userland/capsule_keyring/src/store/types/key_entry.rs    KeyEntry + secure-wipe Drop
  userland/capsule_keyring/src/store/wipe.rs               secure_wipe
```

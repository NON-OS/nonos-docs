# capsule_payment

`capsule_payment` is the settlement rail: it issues a keyring-signed receipt for a paid action, orders
receipts per payer with a nonce, and queues them for an off-capsule drainer to withdraw in batches. It
holds no keys, the signing is the [keyring](keyring.md)'s, and it holds no funds, it records and orders
receipts. Service `payment` on port 4110, reply endpoint `0x1_0000_0010`, capability mask `0x19`. The
source is `userland/capsule_payment/`.

## Contents

- [The server loop](#the-server-loop)
- [The operations](#the-operations)
- [Issuing a receipt](#issuing-a-receipt)
- [The receipt fields](#the-receipt-fields)
- [Draining](#draining)
- [State](#state)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop

`main.rs:28` initializes the heap and calls `server::run` (`src/server/runner.rs:26`) with a 4 KiB buffer
and a stateful `State`, decoding an eight-byte-header frame and dispatching four operations. The reply
endpoint is `0x1_0000_0010`.

## The operations

```
  1  HEALTHCHECK    2  PAY    3  DRAIN_RECEIPTS    4  LIST_TOKENS
```

## Issuing a receipt

`pay` (`src/server/handlers/pay.rs:33`) is the core, and the request is exactly 124 bytes with a fixed
layout:

```
  payload (124 bytes):
      [0..4]    owner_pid (u32 LE)
      [4..8]    wallet_id (u32 LE)
      [8..40]   capsule_id (32)
      [40..60]  publisher (20)         // a 20-byte address
      [60..92]  amount (32)            // 256-bit big-endian
      [92..124] receipt_type (32)

  pay(state, req):
      port = keyring_port()                          else EAGAIN
      require now_ms > 0                              else EINVAL
      nonce = state.next_nonce(owner_pid, wallet_id, publisher, now_ms)   // per-payer, monotonic
      f = ReceiptInput { capsule_id, publisher, amount, nonce,
                         epoch = current_epoch(now_secs), expiry = expiry_at(now_secs), receipt_type }
      signed = keyring.sign_receipt(port, owner_pid, wallet_id, f)         // the keyring holds the key
      if not state.push_receipt(build_record(f, signed)):  EAGAIN          // outbox full
      return signed.struct_hash                                            // 32 bytes
```

The payment capsule marshals the receipt fields, draws a per-payer nonce, and asks the keyring to sign;
it never touches key material. The nonce is keyed by the `(owner_pid, wallet_id, publisher, time)` tuple
and advances per payment, so two receipts from the same payer are ordered.

## The receipt fields

The signed `ReceiptInput` carries the capsule being paid for, the publisher receiving payment, the amount,
the monotonic nonce, an epoch and an expiry derived from the current time (`current_epoch` /
`expiry_at`), and a receipt type. The epoch and expiry are computed from the wall clock at issuance, so a
receipt carries a validity window. The keyring returns a `struct_hash` (the receipt's identity) which the
handler returns to the caller.

## Draining

`drain` (`src/server/handlers/drain.rs`) takes no payload and returns a batch of queued receipt records,
bounded by what fits in the reply buffer (about thirteen 297-byte records per call), clearing them from
the outbox. This is the withdraw side: an off-capsule settlement process periodically drains the accrued
receipts.

## State

`State` (`src/store/types.rs:22`) is a `BTreeMap<[u8; 40], u64>` of payer-tuple to nonce and a
`Vec<Vec<u8>>` outbox of pending records, capped at `MAX_OUTBOX = 1024`. It is in-memory and
append-then-drain; there is no persistence across a restart, and a full outbox rejects new receipts with
`EAGAIN`.

## Security analysis

- **No key custody**: signing is delegated to the [keyring](keyring.md), which holds the key and wipes it
  after use; the payment capsule marshals fields and stores records.
- **Nonce ordering**: per-payer monotonic nonces order receipts and prevent a naive replay of the same
  receipt with the same nonce.
- **Bounded outbox**: the 1024-record cap bounds memory.

The capability mask is `0x19` (`Capsule.mk`, `CAPSULE_REQUIRED_CAPS`), decoding to CoreExec (1), IPC (8),
and Memory (16). This is the correct mask for a capsule that touches neither keys nor funds: it holds no
Crypto, because it delegates all signing to the [keyring](keyring.md) and only marshals the receipt fields;
no Network, because it does not settle on-chain itself but queues records for an off-capsule drainer; no
FileSystem, so the outbox is RAM-only with no persistence; and no hardware capability of any kind. The
least-privilege reading is that a settlement rail is built here without any of the authority you would fear
in one, no key custody, no fund custody, no wire access. Its isolation from the keyring is the design: the
payment capsule reaches the keyring over IPC by resolving its port, and the keyring signs only for the
caller's own wallet, so payment never sees a private key. The honest boundary, stated in the
[gaps](#honest-gaps), is that the inbox has no caller attestation, so the authorization that a wallet
actually consented lives in whether the caller holds the wallet and the keyring's owner-pid check, not in
this capsule.

## Debugging

The service is `payment` on port 4110 (`Capsule.mk`, `service:4110:payment`), reply endpoint
`0x1_0000_0010`. Unlike the core service pools, this capsule is built into the image (the Makefile includes
`userland/capsule_payment/Capsule.mk`) but is *not* spawned by the kernel init spawn plan, so there is no
`[PAYMENT] capsule spawned` marker in the boot fleet the way there is for ramfs or keyring; it is launched
on demand and registers under its manifest endpoint. So the way to tell it is up is not a boot marker but
whether `mk_service_lookup("payment")` resolves, which is exactly what the [installer](installer.md)'s paid
path does: if payment is not reachable that path returns `EAGAIN`, and that `EAGAIN` on a paid install is
the clearest signal the capsule is not registered. Once running, the request-time failure signatures are
`EAGAIN` when the keyring port cannot be resolved (so the receipt cannot be signed) or when the outbox is
full at its 1024-record cap, and `EINVAL` when the wall clock reads zero (no time base for the epoch and
expiry) or the 124-byte `pay` payload is malformed. A `pay` that returns a 32-byte `struct_hash` succeeded;
anything else is one of those two errnos.

## Honest gaps

Stated plainly: the payment capsule has **no caller attestation**, its inbox is a public port, so any
capsule that can reach it can request a payment for any capsule on behalf of any wallet; the guarantee
that the wallet actually authorized the payment lives in whether the caller holds the wallet (and the
keyring signs only for the caller's wallet), not in this capsule. The epoch and expiry are derived from
fixed rules rather than being per-request configurable, and a full outbox drops new receipts with
`EAGAIN`. This capsule is a receipt issuer and queue, not a custody or settlement boundary.

## Source map

```
  userland/capsule_payment/src/server/handlers/pay.rs      the 124-byte pay path, nonce, keyring sign
  userland/capsule_payment/src/server/handlers/drain.rs     the batch withdraw
  userland/capsule_payment/src/fields.rs, record.rs         ReceiptInput, build_record
  userland/capsule_payment/src/sign_call.rs                 the keyring sign call
  userland/capsule_payment/src/store/types.rs               the nonce map + outbox
```

# capsule_market

`capsule_market` serves the application index: it ingests a signed index of available apps, serves their
metadata and releases, and answers whether a given release is ready to install on this machine. It gates
the index on an Ed25519 signature, and it is honest about a development build feature that disables that
gate. Service `market.index` on port 4106, capability mask `0x19`. The source is
`userland/capsule_market/`.

## Contents

- [The server loop](#the-server-loop)
- [The operations](#the-operations)
- [The verifier and the honest stub](#the-verifier-and-the-honest-stub)
- [Ingesting the index](#ingesting-the-index)
- [Install readiness](#install-readiness)
- [State](#state)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop

`main.rs:34` initializes the heap, selects a verifier at compile time (below), and runs the loop
(`src/server/runner.rs:32`) with a large receive buffer (the index blob plus a header) and a 64 KiB reply
buffer, passing the store and the verifier by reference into each handler.

## The operations

Six operations (`src/protocol/ops.rs:17`):

```
  1  LOAD_INDEX    2  LIST_APPS    3  GET_APP    4  GET_RELEASE    5  INSTALL_READY    6  HEALTHCHECK
```

## The verifier and the honest stub

The market gates its index on a signature, and the verifier is chosen at build time by a Cargo feature
(`main.rs:34`):

```
  #[cfg(not(feature = "offline-verify"))]  use CryptoVerifier;   // real: Ed25519
  #[cfg(feature = "offline-verify")]        use RejectAll;        // development stub
```

`CryptoVerifier::verify` (`src/verify/crypto.rs:26`) requires a 64-byte signature and calls
`crypto_ed25519_verify` (through `nonos_libc`, which reaches the [crypto capsule](crypto.md) or a kernel
primitive) over the signed bytes against the operator's public key, returning `Accepted` only on `rc ==
0`. `RejectAll` (`src/verify/reject_all.rs`) always returns `Refused`. The `offline-verify` feature is a
development mode where every index is rejected, and it is documented plainly: a production build uses the
real cryptographic verifier, and the stub exists so the market can be exercised without a signing key.

## Ingesting the index

`LOAD_INDEX` (`src/server/handlers/load_index.rs`) runs the signed index through the verifier and, only if
accepted, replaces the stored index. A stale serial (not monotonic) is `E_STALE`, a bad or untrusted
signature is `E_KEYREJECTED`, and a malformed blob is `E_INVAL`. The verified index becomes the single
`accepted` index the store holds.

## Install readiness

`INSTALL_READY` (`src/server/handlers/install_ready/handle.rs:26`) is the most detailed op: it takes a
`(listing_id, release_id)` pair and returns a six-byte verdict rather than a bare yes/no, so a client
learns *why* an install is or is not ready:

```
  handle(store, body):
      accepted = store.current()                    else E_NODATA
      (listing_id, release_id) = parse_pair(body)   else E_INVAL
      (entry, release) = find_release(accepted.index, listing_id, release_id)   else E_NODATA
      publisher_ok = accepted.publisher_signature_verified(entry, release)
      verdict = evaluate(accepted.signature_verified, release, publisher_ok)
      reply 6 bytes:
          [0] install_ready              [1] index_signature_valid
          [2] package_url_present        [3] publisher_signature_present
          [4] validation_passed          [5] arch_match
```

The verdict reports whether the index and publisher signatures verified, whether the package URL is
present, whether validation passed, and whether the architecture matches this machine, so a client can
tell a signature failure from an architecture mismatch.

## State

The store (`src/store/`) holds a single `accepted` index at a time and replaces it on each successful
`LOAD_INDEX`, along with per-entry flags for whether the index and publisher signatures verified (computed
at ingest).

## Security analysis

- **Signature-gated ingest**: the index is accepted only if its Ed25519 signature verifies against the
  operator key (in a production build).
- **Monotonic serial**: a stale index is rejected, preventing a rollback to an older index.
- **Detailed readiness**: the six-field verdict does not conflate distinct failure reasons.

## Honest gaps

Stated plainly: the market has **no caller attestation** (its inbox is public); the verifier is swapped at
**compile time**, so the reject-all `offline-verify` build cannot verify anything and is for development
only; and the readiness flags for the publisher signature are **pre-computed at ingest** rather than
re-verified per query, so they reflect the state at load time.

## Source map

```
  userland/capsule_market/src/main.rs                              the compile-time verifier selection
  userland/capsule_market/src/verify/crypto.rs                      the real Ed25519 verifier
  userland/capsule_market/src/verify/reject_all.rs                  the offline-verify stub
  userland/capsule_market/src/server/handlers/load_index.rs          verify + store
  userland/capsule_market/src/server/handlers/install_ready/handle.rs the 6-field verdict
  userland/capsule_market/src/store/                                 the single accepted index
```

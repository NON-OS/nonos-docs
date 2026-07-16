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

The capability mask is `0x19` (`Capsule.mk`, `CAPSULE_REQUIRED_CAPS`), decoding to CoreExec (1), IPC (8),
and Memory (16), the same minimal set as the [installer](installer.md) and the [vfs pool](vfs.md). The
least-privilege point is that the market gates its index on a signature but holds no Crypto capability of
its own: `CryptoVerifier::verify` calls `crypto_ed25519_verify` through `nonos_libc`, which reaches the
[crypto capsule](crypto.md) or a kernel primitive over IPC, so the verification runs behind a boundary
rather than in a crypto library linked into the market. It holds no FileSystem (the index arrives over
IPC), no Network (it does not fetch the index itself), and no hardware capability. So the capsule that
decides whether an app is trusted enough to install cannot itself touch a key, a disk, or the wire. Its
isolation is that it holds a single `accepted` index and no per-caller state. The honest boundary, and the
one the page is most careful about, is the compile-time verifier swap: under the `offline-verify` feature
the mask is unchanged but `RejectAll` replaces the real verifier, so the capability posture is identical
between a production build and a development build that cannot verify anything, which is why this is a
build-flag gap the [gaps](#honest-gaps) call out rather than a capability one.

## Debugging

The service is `market.index` on port 4106 (`Capsule.mk`, `service:4106:market.index`), and the capsule is
brought up in the boot fleet at `spawn_plan/core.rs:39` (behind the `nonos-capsule-market` feature) as
`boot::capsule("MARKET", "market", ...)` from `src/security/market_capsule/`. It prints `[MARKET] capsule
spawned` through `capsule_boot::boot` on success, or a `[ERROR]` line with the `SpawnError` (framebuffer
under `NONOS_FBCONSOLE=1`). A present marker means `mk_service_lookup("market.index")` resolves for the
desktop and installer; an absent one means the app index is unavailable. The failure signature that most
often confuses is `E_KEYREJECTED` on `LOAD_INDEX`: in a production build it means the index's Ed25519
signature did not verify against the operator key (traceable one hop into the [crypto](crypto.md) capsule's
`ED25519_VERIFY` returning `EBADMSG`), but in an `offline-verify` development build *every* index is
refused by `RejectAll` regardless of its signature, so `E_KEYREJECTED` on a build you expected to accept a
valid index is the first thing to check against the compile-time feature. The other `LOAD_INDEX` errnos are
`E_STALE` for a non-monotonic serial (a rollback attempt) and `E_INVAL` for a malformed blob, and
`INSTALL_READY` returns a six-byte verdict rather than an errno so a client can separate a signature
failure from an architecture mismatch without guessing.

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

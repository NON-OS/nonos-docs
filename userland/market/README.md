# capsule_market (full reference)

`capsule_market` is the app catalog capsule: it ingests a signed index of available capsules, serves
their metadata and releases over IPC, and answers whether a given release is ready to install on this
machine. It is a read-only index authority. It holds no install logic (that lives in
`capsule_installer`) and no payment logic (that lives in `capsule_payment`), and it never fetches or
installs code itself. Its one job is to decide, behind a signature gate, what the desktop and the
installer are allowed to see and offer.

The whole design turns on one honest boundary: the market gates its index on an Ed25519 signature, and
the verifier that checks that signature is chosen at compile time. A production build links the real
cryptographic verifier; a development build compiled with the `offline-verify` Cargo feature links a
reject-all stub instead, so every signed index is refused. The stub exists so the capsule can be built
and exercised on a kernel image that does not embed `capsule_crypto`, and the page is careful about it
throughout because it is the difference between a build that verifies signatures and one that verifies
nothing. The source is `userland/capsule_market/`.

## Contents

- [Overview and role](#overview-and-role)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview and role

The market is a `no_std`/`no_main` userland service capsule. `_start` initializes the heap, constructs
an empty store, constructs the verifier the build selected, and hands both by reference to the request
loop, which never returns (`src/main.rs:41`, `src/main.rs:46`, `src/main.rs:49`). Everything after that
is request-driven: a caller sends a framed request over the `market.index` IPC endpoint, one handler
runs, and one reply goes back.

Two responsibilities sit behind that loop. The first is ingest: a signed `MarketplaceIndex` blob arrives,
the capsule decodes it, checks that the operator key is trusted, runs the signature through the verifier,
verifies each release's publisher signature, and replaces the single index it holds only if the operator
signature was accepted (`src/ingest/load/load_verified.rs:25`). The second is query: once an index is
accepted, callers can list the catalog, fetch one listing, fetch one release, and ask for an
install-readiness verdict on a `(listing_id, release_id)` pair
(`src/server/handlers/list_apps/handle.rs:28`, `src/server/handlers/get_app/handle.rs:28`,
`src/server/handlers/get_release/handle.rs:24`, `src/server/handlers/install_ready/handle.rs:26`).

The offline-verify swap is the feature to understand first. The default build has no features enabled
(`Cargo.toml:29`); `main.rs` selects `CryptoVerifier` unless `offline-verify` is set, in which case it
selects `RejectAll` (`src/main.rs:34`, `src/main.rs:37`). `RejectAll::verify` returns `Refused`
unconditionally (`src/verify/reject_all.rs:22`), so an `offline-verify` build refuses every index and
serves nothing. That is intentional: the feature comment states the stub keeps install readiness honest
on an image that does not embed `capsule_crypto`, rather than silently accepting unsigned material
(`Cargo.toml:30`).

## Identity

Everything the kernel and the service registry need to name and reach the market comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `market` | `Capsule.mk:7` |
| Service handle | `market` | `Capsule.mk:8` |
| Namespace | `systems.nonos.market` | `Capsule.mk:13` |
| Service endpoint | `service:4106:market.index` | `Capsule.mk:14`, `src/security/market_capsule/spawn.rs:37`, `spawn.rs:38` |
| Reply endpoint | `reply:4107:endpoint.4294967303` | `Capsule.mk:15`, `spawn.rs:39` |
| Capability mask | `0x19` (manifest); `0x18` requested at spawn | `Capsule.mk:17`, `spawn.rs:56` |
| Binary name | `market` | `Capsule.mk:11` |
| Kernel mirror | `src/security/market_capsule` | `Capsule.mk:18` |

The reply endpoint number is not arbitrary: the capsule sends every reply to the kernel reply endpoint
`0x1_0000_0007` (`src/protocol/endpoint.rs:17`), which is decimal `4294967303`, exactly the number in the
reply endpoint name.

The capability mask decomposes against `src/capabilities/types.rs`:

```
  0x0001  CoreExec   bit()  1    types.rs:56
  0x0008  IPC        bit()  8    types.rs:59
  0x0010  Memory     bit() 16    types.rs:60
  ------
  0x0019  = 1 + 8 + 16
```

There is a real discrepancy here worth stating plainly rather than smoothing over. `Capsule.mk:17`
declares `CAPSULE_REQUIRED_CAPS := 0x19` and its comment on `Capsule.mk:16` reads `IPC | Memory = 0x08 |
0x10 = 0x19`, but `0x08 | 0x10` is `0x18`, not `0x19`; the extra bit is CoreExec (`0x01`). The
kernel-side spawn requests exactly `Capability::IPC.bit() | Capability::Memory.bit()`, which is `0x18`,
and requests nothing else (`src/security/market_capsule/spawn.rs:56`). The capsule's own README states
the manifest grants `IPC` and `Memory` and prints the mask as `0x18` (`README.md:40`). So the value the
manifest and attestation are built from (`0x19`, through `--required-caps` and `--capability-mask`,
`nonos-mk/capsule.mk:230`, `nonos-mk/capsule.mk:254`) carries a CoreExec bit that the runtime spawn does
not request, and CoreExec is not added implicitly anywhere in the spawn path. Either way the market holds
no FileSystem, no Network, no Crypto, and no hardware capability; the extra bit is a manifest arithmetic
slip, not a live authority the capsule uses. The prior version of this page decoded `0x19` as CoreExec +
IPC + Memory without noting that the spawn requests only `0x18`.

## Operation reference

Routing is by op number only. The request header carries the op, the loop matches it, and an unknown op
returns `E_INVAL` (`src/server/runner.rs:57`, `src/server/runner.rs:64`). Six ops are defined
(`src/protocol/ops.rs:17`):

| Op | Name | Value | Handler | Reply on success |
|---|---|---|---|---|
| 1 | `OP_LOAD_INDEX` | `1` | `src/server/handlers/load_index.rs:23` | status `0`, no body |
| 2 | `OP_LIST_APPS` | `2` | `src/server/handlers/list_apps/handle.rs:28` | count + per-entry records |
| 3 | `OP_GET_APP` | `3` | `src/server/handlers/get_app/handle.rs:28` | one listing record |
| 4 | `OP_GET_RELEASE` | `4` | `src/server/handlers/get_release/handle.rs:24` | one release record |
| 5 | `OP_INSTALL_READY` | `5` | `src/server/handlers/install_ready/handle.rs:26` | 6-byte verdict |
| 6 | `OP_HEALTHCHECK` | `6` | `src/server/handlers/health.rs:20` | status `0`, no body |

Every reply, success or error, starts with the 20-byte response header (magic, version, op, flags, a
zeroed field, request id, payload length) followed by a 4-byte little-endian status word
(`src/protocol/encode/encode_response_header.rs:19`, `src/protocol/encode/write_status.rs:17`). An error
reply is header plus status only (`src/server/error/reply_status.rs:23`); a data reply appends the body
after the status (`src/server/payload/reply_with_body.rs:23`).

### LOAD_INDEX (op 1)

`handle` reads the store's last accepted serial, runs the blob through `load_verified`, and on success
installs the returned index and replies status `0` (`src/server/handlers/load_index.rs:30`). The
verification pipeline, in order (`src/ingest/load/load_verified.rs:25`):

1. Decode the blob with the marketplace ABI; a decode failure is `Malformed`
   (`load_verified.rs:30`, `userland/marketplace_abi/src/codec/decode_index.rs:35`).
2. Reject a non-monotonic serial: if the new serial is at or below the last accepted serial and a prior
   index exists, that is `StaleSerial` (`load_verified.rs:31`). The first load, against serial `0`, is
   always allowed.
3. Reject an untrusted operator key: the index's `operator_pubkey` must be one of the baked trusted
   operators, else `UntrustedOperator` (`load_verified.rs:34`, `src/bootstrap_trust/check.rs:19`).
4. Run the operator signature through the verifier over the decoded signed bytes; anything but `Accepted`
   is `SignatureRefused` (`load_verified.rs:37`).
5. Verify each release's publisher signature and record a per-release boolean vector
   (`load_verified.rs:45`, `src/ingest/load/verify_publisher_signatures.rs:25`).

The handler maps those errors to errnos: `Malformed` to `E_INVAL`, `StaleSerial` to `E_STALE`, and both
`SignatureRefused` and `UntrustedOperator` to `E_KEYREJECTED`
(`src/server/handlers/load_index.rs:40`). The store then holds exactly one accepted index, its operator
signature flag, and the publisher flag vector (`src/store/state/install.rs:25`).

### LIST_APPS (op 2)

Requires an accepted index, else `E_NODATA` (`list_apps/handle.rs:29`). The body is a little-endian entry
count followed, per entry, by the length-prefixed `listing_id`, the raw `capsule_id`, the length-prefixed
`name`, and a single readiness byte that is `1` only if at least one of the entry's releases evaluates as
install-ready (`list_apps/handle.rs:34`). If the assembled body does not fit the reply slot the handler
returns `E_MSGSIZE` (`list_apps/handle.rs:49`).

### GET_APP (op 3)

Requires an accepted index (`E_NODATA`) and a single length-prefixed `listing_id` in the body (`E_INVAL`
on a malformed length prefix) (`get_app/handle.rs:29`, `get_app/handle.rs:33`). An unknown listing is
`E_NODATA` (`get_app/handle.rs:39`). The reply carries the listing's id, capsule id, name, publisher
name, publisher pubkey, description, and release count (`get_app/handle.rs:42`), or `E_MSGSIZE` if it
overflows (`get_app/handle.rs:52`).

### GET_RELEASE (op 4)

Requires an accepted index (`E_NODATA`) and a `(listing_id, release_id)` pair, each length-prefixed
(`E_INVAL` on a bad pair) (`get_release/handle.rs:25`, `get_release/handle.rs:29`). A pair that names no
release is `E_NODATA` (`get_release/handle.rs:41`). On a hit the reply is the encoded release record
(`get_release/handle.rs:43`), or `E_MSGSIZE` on overflow (`get_release/handle.rs:47`).

### INSTALL_READY (op 5)

The most detailed op. It takes a `(listing_id, release_id)` pair and returns a six-byte verdict rather
than a bare yes/no, so a caller learns why an install is or is not ready
(`src/server/handlers/install_ready/handle.rs:26`):

```
  handle(store, body):
      accepted = store.current()                       else E_NODATA
      (listing_id, release_id) = parse_pair(body)      else E_INVAL
      (entry_i, release_i, release) =
          find_release(accepted.index, listing_id, release_id)   else E_NODATA
      publisher_ok = accepted.publisher_signature_verified(entry_i, release_i)
      verdict = evaluate(accepted.signature_verified, release, publisher_ok)
      reply 6 bytes:
          [0] install_ready              [1] index_signature_valid
          [2] package_url_present        [3] publisher_signature_present
          [4] validation_passed          [5] arch_match
```

The six bytes are written in that fixed order and the reply length is `READINESS_LEN = 6`
(`install_ready/handle.rs:46`, `src/server/handlers/install_ready/constants.rs:17`). `evaluate` computes
the fields from the release and the two verified flags (`src/install_ready/checks.rs:23`):
`index_signature_valid` mirrors the operator signature flag; `validation_passed` requires
`ValidationStatus::Validated`; `package_url_present` in the reply folds in that the package hash and
manifest hash are non-zero as well as the URL being non-empty; `publisher_signature_present` is the
per-release publisher flag; and `arch_match` in the reply folds the running-arch match together with
kernel-ABI compatibility (`checks.rs:33`, `checks.rs:48`, `checks.rs:51`). The single `install_ready`
byte is the conjunction of all of those plus the package and manifest hashes being present
(`checks.rs:36`). The running arch triple is compile-time selected and a build for an unknown arch is a
hard `compile_error!` (`src/install_ready/arch.rs:17`, `arch.rs:27`).

### HEALTHCHECK (op 6)

Replies status `0` with no body and touches no state (`src/server/handlers/health.rs:20`). It is a
liveness probe.

### Errors

Every errno the capsule can return (`src/protocol/errno.rs`):

| Errno | Value | Meaning | Raised by |
|---|---|---|---|
| `E_INVAL` | `-22` | malformed request, bad length prefix, or unknown op | `errno.rs:17`, `runner.rs:44`, `runner.rs:64` |
| `E_NODATA` | `-61` | no accepted index yet, or the named listing/release is absent | `errno.rs:18`, `install_ready/handle.rs:29` |
| `E_STALE` | `-116` | index serial is not strictly greater than the last accepted | `errno.rs:20`, `load_index.rs:41` |
| `E_MSGSIZE` | `-90` | request body exceeds the receive buffer, or a reply overflows the slot | `errno.rs:21`, `runner.rs:52` |
| `E_KEYREJECTED` | `-129` | operator signature refused or operator key untrusted | `errno.rs:19`, `load_index.rs:42`, `load_index.rs:43` |

## Architecture and lifecycle

The crate has seven top-level modules (`src/main.rs:22`): `bootstrap_trust` (the baked trusted operator
keys), `ingest` (blob decode plus the verification pipeline), `install_ready` (the readiness evaluator
and the running-arch triple), `protocol` (the wire header, ops, errnos, and codecs), `server` (the loop
and handlers), `store` (the single accepted index), and `verify` (the verifier trait and its two
implementations).

The store is a single `Option<Accepted>` (`src/store/state/store_type.rs:19`), where `Accepted` is the
index, the operator signature flag, and the flat per-release publisher flag vector
(`src/store/state/accepted.rs:22`). `current` returns the accepted index or `None`
(`src/store/state/current.rs:20`); `last_serial` returns its serial or `0` when empty
(`src/store/state/last_serial.rs:20`); `install` replaces the whole accepted value
(`src/store/state/install.rs:25`). `publisher_signature_verified` flattens an `(entry, release)` index
pair into the linear vector to look up one release's flag, returning `false` for an out-of-range index
(`src/store/state/publisher_signature_verified.rs:20`).

The verify path is a small trait. `Verifier::verify` takes signed bytes, a signature, and a 32-byte
pubkey and returns `Verdict::Accepted` or `Verdict::Refused` (`src/verify/trait_def.rs:23`).
`CryptoVerifier` rejects any signature that is not 64 bytes, then calls `crypto_ed25519_verify` through
`nonos_libc`, accepting only on `rc == 0` (`src/verify/crypto.rs:26`). `RejectAll` returns `Refused`
unconditionally (`src/verify/reject_all.rs:22`). The same trait object verifies both the operator
signature and every publisher signature, so the offline stub disables both at once
(`src/ingest/load/verify_publisher_signatures.rs:38`).

Lifecycle:

1. The kernel spawns the capsule at boot through the boot fleet, behind the `nonos-capsule-market`
   feature: `spawn_market` calls `boot::capsule("MARKET", "market", spawn_market_capsule, shared_state)`
   (`src/userspace/init/spawn_plan/core.rs:36`, `spawn_plan/core.rs:39`). When the feature is off,
   `spawn_market` is a no-op (`spawn_plan/core.rs:42`).
2. `spawn_market_capsule` decodes the baked trust anchor and hands the embedded ELF, id cert, manifest,
   and attestation trailer to `spawn_verified`, registering `market.index` on service port 4106 with a
   reply on 4107 and requesting `IPC | Memory` (`src/security/market_capsule/spawn.rs:42`,
   `spawn.rs:56`, `spawn.rs:59`). On success it records the pid alive (`spawn.rs:60`).
3. The boot helper logs `[MARKET] capsule spawned` on success or an `[ERROR]` line on failure
   (`src/userspace/init/capsule_boot/run.rs:29`, `capsule_boot/run.rs:32`).
4. Inside the capsule, `_start` sets up the heap and enters `server::run`, which loops on `mk_ipc_recv`
   with a receive buffer sized to the max index blob plus a header and a 64 KiB transmit buffer, decodes
   each request, bounds-checks its declared payload against the bytes received, dispatches by op, and
   sends one reply (`src/main.rs:49`, `src/server/runner.rs:32`, `src/protocol/limits.rs:20`).

## Protocol and IPC

The wire is a fixed 20-byte header. The request header is magic `0x4E4D_4B54`, version `1`, op, flags, a
reserved 2-byte hole, a request id, and a payload length (`src/protocol/header.rs:17`,
`src/protocol/decode.rs:19`). `decode_request` rejects anything shorter than the header, a wrong magic, or
a wrong version (`decode.rs:20`). The runner then requires the declared payload to fit inside the bytes
actually received, replying `E_MSGSIZE` otherwise (`src/server/runner.rs:51`). A response reuses the same
header with the reserved field zeroed, the request's op, flags, and id echoed back, and the payload
length set to the status word plus any body (`src/protocol/encode/encode_response_header.rs:19`).

The capsule speaks two IPC verbs and no others. It receives on inbox `0` with `mk_ipc_recv`
(`src/server/runner.rs:36`) and sends every reply to the kernel reply endpoint `0x1_0000_0007` with
`mk_ipc_send` (`src/server/error/reply_status.rs:26`, `src/server/payload/reply_with_body.rs:28`).

For signature checks it calls one downstream service. `CryptoVerifier::verify` invokes
`crypto_ed25519_verify` through `nonos_libc` (`src/verify/crypto.rs:31`). The kernel routes that call to
`capsule_crypto` through the `CryptoEd25519Verify` syscall path, which is why the market needs no Crypto
capability of its own: the crypto authority lives behind that boundary (`Capsule.mk:2`, `Cargo.toml:30`,
`src/security/market_capsule/spawn.rs:17`). It calls the installer or the payment service for nothing;
those are separate capsules and separate concerns (`Cargo.toml:4`).

The kernel-side client of this capsule is caller-gated. `gate_call` requires the calling pid to hold
`CAP_APPS` before any request is forwarded, because the marketplace is the apps-discovery surface
(`src/security/market_capsule/capability.rs:25`, `capability.rs:30`). That gate lives in the kernel
mirror, not in the capsule; the capsule's own inbox performs no caller attestation.

## Security analysis

The market decides what the desktop and the installer are allowed to see, so its trustworthiness is the
whole point, and its authority is deliberately tiny. The kernel spawns it with `IPC | Memory` and nothing
else (`src/security/market_capsule/spawn.rs:56`). It holds no FileSystem, so the index cannot arrive off
a disk it reads; no Network, so it cannot fetch the index itself; no Crypto, so it cannot hold or use a
key directly; and no driver, MMIO, IRQ, DMA, or PIO capability. The capsule that decides whether an app
is trusted enough to install cannot itself touch a key, a disk, or the wire.

The ingest gate is layered and each layer is a distinct refusal (`src/ingest/load/load_verified.rs:25`):

- Trusted operator. The operator pubkey must match a baked key before any signature is even checked
  (`src/bootstrap_trust/check.rs:19`), and the trusted set is a compiled-in constant with a single
  current operator (`src/bootstrap_trust/keys.rs:17`, `keys.rs:22`). A stranger's key is `E_KEYREJECTED`
  regardless of how well-formed its signature is.
- Signature-gated index. The operator signature must verify over the decoded signed bytes, and only an
  `Accepted` verdict lets the index replace the stored one (`load_verified.rs:37`, `load_verified.rs:42`).
- Monotonic serial. A serial at or below the last accepted one is `E_STALE`, so an attacker cannot roll
  the catalog back to an older, weaker index once a newer one has been accepted (`load_verified.rs:31`).
- Per-release publisher signatures. Each release's publisher signature is checked at ingest against the
  entry's publisher key, with a zero publisher key or a wrong-length signature counting as unverified
  rather than trusted (`src/ingest/load/verify_publisher_signatures.rs:32`).

The readiness verdict does not conflate failure reasons: a caller can tell a signature failure from an
architecture mismatch from a missing package URL, because each is its own byte
(`src/server/handlers/install_ready/handle.rs:46`). That matters because the installer downstream keys
its decision off these fields.

The real-verifier-versus-stub swap is the boundary this page is most careful about, and it is a
build-flag property, not a capability one. `CryptoVerifier` routes real Ed25519 checks to the crypto
capsule (`src/verify/crypto.rs:31`); `RejectAll` returns `Refused` for everything
(`src/verify/reject_all.rs:22`). The choice is `#[cfg]`-gated at compile time (`src/main.rs:34`,
`src/main.rs:37`) and the `offline-verify` feature is off by default (`Cargo.toml:29`). The capability
mask is identical between the two builds, so a development image that cannot verify a single signature has
the exact same authority posture as a production one; the only difference is that the offline build
refuses every index rather than accepting a valid one. That is why the gap below is a build-flag gap, not
an authority gap.

Honest gaps:

- The capsule's own inbox performs no caller attestation; it answers whoever reaches it. The `CAP_APPS`
  gate that restricts callers lives in the kernel-side client (`src/security/market_capsule/capability.rs:30`),
  not in the capsule, so anyone who can send to the endpoint directly is served.
- The verifier is swapped at compile time, and an `offline-verify` build cannot verify anything; it is a
  development build only (`src/main.rs:37`, `src/verify/reject_all.rs:22`).
- The publisher-signature flags are computed once at ingest and stored, then read back per query rather
  than re-verified on each `install_ready` call, so they reflect the state at load time
  (`src/store/state/publisher_signature_verified.rs:20`, `src/ingest/load/verify_publisher_signatures.rs:25`).
- The manifest capability mask (`0x19`) carries a CoreExec bit that the runtime spawn (`0x18`) does not
  request; the value is a `Capsule.mk` arithmetic slip rather than a live authority (see
  [Identity](#identity)).

## How to contribute

The source lives at `userland/capsule_market/`. The wire and codecs are under `src/protocol/`, the loop
and handlers under `src/server/`, the ingest pipeline under `src/ingest/`, the readiness evaluator under
`src/install_ready/`, the store under `src/store/`, the verifier under `src/verify/`, and the baked
operator keys under `src/bootstrap_trust/`.

To add a new op:

1. Add its discriminant in `src/protocol/ops.rs:17`. Op numbers are the only routing key, so a new op is
   a new constant.
2. Write the handler as its own module under `src/server/handlers/`, following the existing shape: pull
   the accepted index (or return `E_NODATA`), parse the body with a length-prefix helper (or return
   `E_INVAL`), assemble the reply into the transmit slot with `body_slot`, and send it with
   `reply_with_body` (`src/server/handlers/get_app/handle.rs:28` is the shortest full example). Re-export
   it from `src/server/handlers/mod.rs:17`.
3. Wire the op into the dispatch match in `src/server/runner.rs:57`.

The verifier is a trait, so an alternate verification backend is a new type implementing
`Verifier` and a matching `#[cfg]` arm in `src/main.rs:34`; do not verify signatures inline in a handler.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_market/Capsule.mk:20`):

```
  make nonos-mk-market                build the capsule ELF
  make nonos-mk-market-sign           produce the id cert, manifest, and attestation trailer
  make nonos-mk-market-verify         verify the signed manifest against the trust anchor
  make nonos-mk-check-market-keys     check the per-capsule signing keys exist
```

For a running kernel that includes the market, `make nonos-mk-market-prod` builds the profile under the
`microkernel-market` kernel feature (`Makefile:930`). Two market-specific host targets support the
kernel-side smoke path: `make nonos-mk-market-smoke` builds the capsule under a smoketest-trust key
(`Makefile:760`) and `make nonos-mk-market-fixtures` generates the signed index fixtures the smoke
embeds (`Makefile:798`), both driven through the host `marketplace-index` CLI
(`Makefile:780`). The capsule depends on the `marketplace_abi` rlib, which the Makefile rebuilds before
the market when the ABI changes (`Makefile:753`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns errors as errnos through `reply_status`, never a panic;
the release profile is `panic = "abort"`, `Cargo.toml:37`); modular files, one unit per file, with
`mod.rs` used only for re-exports; and the AGPL header at the top of every source file, matching the
header on every existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints `[MARKET]
capsule spawned` from the boot log (`src/userspace/init/capsule_boot/run.rs:29`); an absent line means
the capsule never started, usually a signature, manifest, or capability failure, and the error path
prints an `[ERROR]` line instead (`capsule_boot/run.rs:32`). The kernel-side spawn also carries a
`[MARKET-DEBUG] load_elf_executable error:` tag on an ELF load failure
(`src/security/market_capsule/spawn.rs:57`). A present marker means `market.index` resolves on port 4106
for the desktop and the installer; an absent one means the app catalog is unavailable.

Failure modes and where to look:

- `E_KEYREJECTED` on `LOAD_INDEX`. In a production build this means the operator signature did not verify
  against the trusted operator key, or the operator key is not in the trusted set at all
  (`src/server/handlers/load_index.rs:42`, `load_index.rs:43`). But in an `offline-verify` development
  build every index is refused by `RejectAll` no matter what its signature is
  (`src/verify/reject_all.rs:22`), so `E_KEYREJECTED` on a build you expected to accept a valid index is
  the first thing to check against the compile-time feature (`src/main.rs:37`, `Cargo.toml:29`).
- `E_STALE` on `LOAD_INDEX`. The new index's serial is not strictly greater than the last accepted one, a
  rollback attempt (`load_index.rs:41`, `src/ingest/load/load_verified.rs:31`). The very first load runs
  against serial `0` and is allowed; a repeat of the same serial is stale.
- `E_INVAL` on `LOAD_INDEX`. The blob did not decode (`load_index.rs:40`), distinct from a signature
  failure. On the query ops, `E_INVAL` instead means a malformed length prefix in the request body
  (`src/server/handlers/get_app/handle.rs:35`).
- `E_NODATA` on any query. No index has been accepted yet (`install_ready/handle.rs:29`), or the named
  listing or release is not in the accepted index (`get_release/handle.rs:41`). Check that a `LOAD_INDEX`
  succeeded before assuming a lookup bug.
- An install that looks ready but the installer refuses it. `INSTALL_READY` returns six independent bytes
  (`install_ready/handle.rs:46`); read them individually. A zero in `index_signature_valid` versus a zero
  in `arch_match` versus a zero in `publisher_signature_present` isolate three different causes that a
  bare yes/no would hide.

## Source map

```
  src/main.rs                                   _start -> server::run; the compile-time verifier selection
  src/verify/trait_def.rs                       the Verifier trait and Verdict
  src/verify/crypto.rs                          the real Ed25519 verifier (routes to capsule_crypto)
  src/verify/reject_all.rs                      the offline-verify reject-all stub
  src/bootstrap_trust/keys.rs                   the baked trusted operator keys
  src/ingest/load/load_verified.rs              decode + serial + trust + signature + publisher pipeline
  src/ingest/load/verify_publisher_signatures.rs per-release publisher signature check
  src/install_ready/checks.rs                   the readiness evaluator (six fields)
  src/install_ready/arch.rs                     the compile-time running-arch triple
  src/protocol/ops.rs                           the six op discriminants
  src/protocol/errno.rs                         the errno set
  src/protocol/header.rs                        the 20-byte wire header
  src/server/runner.rs                          the recv/decode/dispatch/reply loop
  src/server/handlers/                          load_index, list_apps, get_app, get_release, install_ready, health
  src/store/state/                              the single accepted index and its flags
  Capsule.mk                                    slug, handle, ports, capability mask, kernel mirror
  src/security/market_capsule/                  the kernel-side embed, verified spawn, and CAP_APPS client gate
  src/userspace/init/spawn_plan/core.rs         the boot-fleet spawn entry (nonos-capsule-market)
  userland/marketplace_abi/                     the shared index and release codec the capsule decodes
  nonos-mk/capsule.mk                           the generated nonos-mk-market[-sign|-verify] targets
```

Every reference above is verified against those trees.

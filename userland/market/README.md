# The Market Capsule

`capsule_market` is the app catalog: a signed userland service that ingests one signed index of available
capsules, serves their metadata and releases over IPC, and answers whether a given release is ready to
install on this machine. It is a read-only index authority. It holds no install logic (that lives in
[capsule_installer](../installer/README.md)) and no payment logic, and it never fetches or installs code
itself. Its one job is to decide, behind a signature gate, what the desktop and the installer are allowed
to see and offer.

The whole design turns on one honest boundary. The market gates its index on an Ed25519 signature, and the
verifier that checks that signature is chosen at compile time. A production build links the real
cryptographic verifier; a development build compiled with the `offline-verify` Cargo feature links a
reject-all stub instead, so every signed index is refused. This documentation mirrors the source one page
per pillar so a page can be read beside the folder it describes, and it is careful about the
verifier-versus-stub split throughout, because that split is the difference between a build that verifies
signatures and one that verifies nothing.

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
| Capability mask | `0x39` (manifest) | `Capsule.mk:17` |
| Binary name | `market` | `Capsule.mk:11` |
| Kernel mirror | `src/security/market_capsule` | `Capsule.mk:18` |

The reply endpoint number is not arbitrary. The capsule sends every reply to the kernel reply endpoint
`0x1_0000_0007` (`src/protocol/endpoint.rs:17`), which is decimal `4294967303`, exactly the number in the
reply endpoint name.

The capability mask decomposes against `src/capabilities/types/defs.rs`:

```
  0x0001  CoreExec   = 1
  0x0008  IPC        = 8
  0x0010  Memory     = 16
  0x0020  Crypto     = 32
  ------
  0x0039  = 1 + 8 + 16 + 32
```

`Crypto` is in the mask because the release-signature check calls `crypto_ed25519_verify`, whose syscall
is gated on `Capability::Crypto` at the contract layer; without it that verification is denied
(`userland/capsule_market/Capsule.mk:3`). There is a subtlety worth stating plainly. The manifest declares
`CAPSULE_REQUIRED_CAPS := 0x39` (CoreExec, IPC, Memory, Crypto), while the kernel-side spawn passes
`requested_caps = IPC | Memory | Crypto`, which is `0x38` and omits CoreExec
(`src/security/market_capsule/spawn.rs:56`). That difference does not attenuate the capsule: `requested_caps`
bounds only optional caps, and the market declares no optional caps, so the installed set is the manifest's
required `0x39`, CoreExec included (the install rule is `required | (optional & granted)`, see the
[userland model](../README.md)). The market holds no FileSystem, Network, or hardware capability; `Crypto`
is the only sensitive bit, and it exists solely for the Ed25519 index-signature check.

## The pillars

The source under `userland/capsule_market/src/` is seven top-level modules (`src/main.rs:22`). They group
into three pillars, and the documentation is one page each. An index blob arrives, the verification pillar
either accepts it into the store or refuses it, and the protocol pillar then serves queries against that
store, one of which is the readiness verdict.

```
  ingest + verify + bootstrap_trust   ->   store   ->   protocol + server
  the signature gate                       the one       the wire, the ops,
  that guards the catalog                  accepted      and the handlers that
                                           index         answer queries
```

| Page | Mirrors | What it covers |
|------|---------|----------------|
| [protocol.md](protocol.md) | `src/protocol/`, `src/server/` | The 20-byte wire header, the six ops and their errnos, the receive/decode/dispatch/reply loop, and each handler's request and reply shape. |
| [verification.md](verification.md) | `src/verify/`, `src/ingest/`, `src/bootstrap_trust/` | The verifier trait and its two implementations, the real-verifier-versus-stub swap, the trusted-operator gate, the monotonic-serial check, and the per-release publisher signatures. |
| [readiness.md](readiness.md) | `src/install_ready/`, `src/store/` | The six-byte install-readiness verdict, the fields it decomposes into, the compile-time running-arch triple, and the single accepted index the store holds. |
| [contributing.md](contributing.md) | the whole tree | Where to work, how to add an op, the build and sign steps, and the code standards. |
| [debugging.md](debugging.md) | runtime | The boot marker, the errno failure modes, and where to look when ingest refuses an index or a query returns nothing. |

## Lifecycle

1. The kernel spawns the capsule at boot through the boot fleet, behind the `nonos-capsule-market`
   feature: `spawn_market` calls `boot::capsule("MARKET", "market", spawn_market_capsule, shared_state)`
   (`src/userspace/init/spawn_plan/core.rs:36`, `spawn_plan/core.rs:39`). When the feature is off,
   `spawn_market` is a no-op (`spawn_plan/core.rs:42`).
2. `spawn_market_capsule` decodes the baked trust anchor and hands the embedded ELF, id cert, manifest,
   and attestation trailer to `spawn_verified`, registering `market.index` on service port 4106 with a
   reply on 4107 and requesting `IPC | Memory` (`src/security/market_capsule/spawn.rs:42`, `spawn.rs:56`,
   `spawn.rs:59`). On success it records the pid alive (`spawn.rs:60`).
3. The boot helper logs `[MARKET] capsule spawned` on success or an error line on failure
   (`src/userspace/init/capsule_boot/run.rs:29`, `capsule_boot/run.rs:32`).
4. Inside the capsule, `_start` initializes the heap, constructs an empty store, constructs the verifier
   the build selected, and enters `server::run`, which never returns (`src/main.rs:41`, `src/main.rs:46`,
   `src/main.rs:49`). Everything after that is request-driven: a caller sends a framed request over
   `market.index`, one handler runs, and one reply goes back.

The market registers no authority of its own beyond `IPC | Memory`. It holds no FileSystem, so the index
cannot arrive off a disk it reads; no Network, so it cannot fetch the index itself; and no Crypto, so it
cannot hold or use a key directly. The capsule that decides whether an app is trusted enough to install
cannot itself touch a key, a disk, or the wire. The kernel-side client that forwards requests to the
capsule is caller-gated on `CAP_APPS` (`src/security/market_capsule/capability.rs:30`); the capsule's own
inbox performs no caller attestation and answers whoever reaches it.

## Source map

```
  userland/capsule_market/src/main.rs         _start -> server::run; the compile-time verifier selection
  userland/capsule_market/src/protocol/       the wire header, ops, errnos, and codecs
  userland/capsule_market/src/server/         the recv/decode/dispatch/reply loop and the handlers
  userland/capsule_market/src/ingest/         blob decode plus the verification pipeline
  userland/capsule_market/src/verify/         the Verifier trait and its two implementations
  userland/capsule_market/src/bootstrap_trust/ the baked trusted operator keys
  userland/capsule_market/src/install_ready/  the readiness evaluator and the running-arch triple
  userland/capsule_market/src/store/          the single accepted index and its flags
  userland/capsule_market/Capsule.mk          slug, handle, ports, capability mask, kernel mirror
  userland/marketplace_abi/                   the shared index and release codec the capsule decodes
  src/security/market_capsule/                the kernel-side embed, verified spawn, and CAP_APPS client gate
  src/capabilities/types/defs.rs              the capability bits behind the mask
```

Every reference above is verified against those trees.

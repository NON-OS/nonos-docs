# capsule_entropy

`capsule_entropy` is the userland random-bytes service: the source that the kernel's `CryptoRandom`
syscall path draws from when a capsule asks for random bytes. It is deliberately thin. It draws bytes
directly from the CPU hardware random generator (`RDRAND`), serves them under a per-request cap, and
keeps four counters so the entropy path is observable. It holds no software CSPRNG pool of its own; it
is a monitored pass-through to the hardware source, which is why it is honest to call it a wrapper over
`RDRAND` rather than a mixing pool. The name "pool" in the code refers to the accounting object, not to
a mixed entropy buffer.

The capsule registers as service `entropy_pool` on port 4100 with a reply endpoint on 4101, and its
capability mask is `0x39` (`userland/capsule_entropy/Capsule.mk:17`). The source is
`userland/capsule_entropy/`, and the kernel-side mirror that embeds, spawns, and calls it lives at
`src/security/entropy_capsule/`.

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

The capsule replaces the old kernel-resident entropy engine: user-facing random requests are meant to
route through this capsule rather than a kernel RNG shim (`userland/capsule_entropy/Cargo.toml:4`). The
only in-tree caller is the kernel's `CryptoRandom` syscall handler, which gates on `CAP_CRYPTO`, then
asks the capsule for bytes over IPC (`src/syscall/dispatch/crypto/random.rs:31`,
`src/syscall/dispatch/crypto/random.rs:39`). When the capsule answers, its bytes are what the caller
receives; when the capsule is unavailable, the syscall falls back to the kernel hardware RNG so a
missing capsule never starves a caller (`src/syscall/dispatch/crypto/random.rs:43`). That fallback is
worth stating up front, because it means the capsule is the primary path but not the only one.

The capsule itself does no policy. It owns a request loop, a small accounting object it calls `Pool`,
and a handful of stateless handlers. It never persists anything: the counters live in memory and reset
on restart (`userland/capsule_entropy/src/pool/types.rs:26`).

## Identity

Everything the kernel and the service registry need to name and reach the capsule comes from its
`Capsule.mk` and the kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `entropy` | `Capsule.mk:7` |
| Service handle | `entropy` | `Capsule.mk:8` |
| Namespace | `systems.nonos.entropy` | `Capsule.mk:13` |
| Service endpoint | `service:4100:entropy_pool` | `Capsule.mk:14`, `src/security/entropy_capsule/spawn.rs:31` |
| Reply endpoint | `reply:4101:endpoint.4294967299` | `Capsule.mk:15`, `src/security/entropy_capsule/spawn.rs:33` |
| Reply inbox name | `endpoint.4294967299` (= `0x1_0000_0003`) | `src/security/entropy_capsule/client/transport.rs:27`, `src/security/entropy_capsule/protocol.rs` |
| Capability mask | `0x39` | `Capsule.mk:17` |
| Binary name | `entropy` | `Capsule.mk:11` |
| Kernel mirror | `src/security/entropy_capsule` | `Capsule.mk:18` |

The service name the capsule serves on the wire is `entropy_pool` on port 4100; the reply endpoint the
capsule sends to is the kernel-owned inbox `endpoint.4294967299`, whose numeric form `0x1_0000_0003`
is the constant `KERNEL_REPLY_ENDPOINT` the capsule targets (`src/protocol/types.rs:42`,
`src/server/runner.rs:43`). The reply port itself is 4101 (`src/security/entropy_capsule/spawn.rs:33`).

The mask `0x39` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x08  IPC       bit()   8   types.rs:59
  0x10  Memory    bit()  16   types.rs:60
  0x20  Crypto    bit()  32   types.rs:62
  ------
  0x39  = 8 + 16 + 32
```

The kernel spawn path requests exactly those three bits and no others: `Capability::IPC.bit() |
Capability::Memory.bit() | Capability::Crypto.bit()` (`src/security/entropy_capsule/spawn.rs:54`). There
is no `CoreExec`, no `Network`, no `FileSystem`, and no hardware, driver, MMIO, IRQ, DMA, or PIO bit.
The reason the capsule that is the entropy authority does not itself hold an entropy capability is
recorded in its own `Capsule.mk`: the capsule is the authority, so callers carry the entropy bit and
reach the pool through IPC, while the capsule needs only IPC for `mk_ipc_*`, Memory for its heap, and
Crypto for the RNG primitive it consumes (`Capsule.mk:1`). The Crypto bit is held because `RDRAND`
sits on the crypto/random path; the instruction is executed in the capsule's own context under a
`target_feature` gate, not by claiming a device through the broker, so no device claim, MMIO map, or
IRQ binding is involved and none is granted.

## Operation reference

The wire frame is a 20-byte header followed by a payload. The decode validates the magic, the version,
and that the declared payload length fits the frame before dispatch (`src/protocol/decode.rs:29`). Four
operations are defined (`src/protocol/types.rs:30`), dispatched in `src/server/dispatch.rs:25`; an
unknown op returns `EINVAL` (`src/server/dispatch.rs:31`).

| Op | Opcode | Handler | Request payload | Reply payload (after status) |
|---|---|---|---|---|
| GET_RANDOM | 1 | `get_random` | `u32` length (LE) | `length` random bytes |
| GET_STATS | 2 | `get_stats` | none | 32-byte stats blob |
| RESEED | 3 | `reseed` | `u32` len (LE) + `len` bytes | none |
| HEALTHCHECK | 4 | `healthcheck` | none | none |

Opcodes are `OP_GET_RANDOM = 1`, `OP_GET_STATS = 2`, `OP_RESEED = 3`, `OP_HEALTHCHECK = 4`
(`src/protocol/types.rs:30`). Every reply carries an `i32` status in the first four bytes of its
payload, little-endian; a zero status means success (`src/protocol/encode.rs:25`).

### GET_RANDOM (op 1)

`get_random` (`src/server/handlers/getrandom.rs:27`) parses the length, caps it, fills, and checks the
result exactly:

```
  get_random(pool, req):
      if payload < 4:                 EINVAL          // no length word
      length = u32(payload[0..4])
      if length > MAX_RANDOM_BYTES:   EMSGSIZE        // 4096 ceiling
      out = vec![0; length]
      n = pool.fill(out)
      if n < 0 or n != length:        EIO             // hardware source failure
      return out                                      // status 0 + length bytes
```

The size ceiling is `MAX_RANDOM_BYTES = 4096` (`src/protocol/types.rs:36`), checked at
`getrandom.rs:32`. `pool.fill` (`src/pool/fill.rs:42`) itself re-clamps to the same cap, increments
the `requests` counter before the attempt, returns `-5` and increments `source_failures` if the
hardware fill gave up, and otherwise adds the served count to `bytes_served` and returns it. So a
successful request returns exactly the requested number of hardware-random bytes, and a hardware
failure is a clean `EIO` (value `-5`, `src/protocol/errno.rs:17`).

### GET_STATS (op 2)

`get_stats` (`src/server/handlers/getstats.rs:23`) reads no untrusted input. It takes a snapshot of the
four counters (`src/pool/snapshot.rs:21`) and encodes them as a fixed 32-byte little-endian blob:
`uptime_requests` at bytes 0..8, `bytes_served` at 8..16, `last_reseed_request` at 16..24, and
`source_failures` at 24..32 (`src/pool/encode_stats.rs:18`). Status is always 0.

### RESEED (op 3)

`reseed` (`src/server/handlers/reseed.rs:26`) bounds-checks the supplied entropy, records the reseed
for observability, and acknowledges:

```
  reseed(pool, req):
      if payload < 4:                       EINVAL
      length = u32(payload[0..4])
      if length > MAX_RESEED_BYTES:         EINVAL    // 256 ceiling
      if 4 + length != payload.len():       EINVAL    // declared vs actual
      pool.record_reseed()                            // bump last_reseed_request
      return ()                                       // status 0, empty body
```

The ceiling is `MAX_RESEED_BYTES = 256` (`src/protocol/types.rs:37`), checked at `reseed.rs:31`, and
the handler also rejects a length that does not match the actual payload size (`reseed.rs:34`). It does
not mix the supplied bytes into any state, because there is no userland CSPRNG state to mix into:
`record_reseed` only increments the `last_reseed_request` counter (`src/pool/record_reseed.rs:21`). So
`RESEED` on this capsule is an observability breadcrumb, not a mixing operation.

### HEALTHCHECK (op 4)

`healthcheck` (`src/server/handlers/healthcheck.rs:24`) takes no input and returns an empty
success reply. Reaching it proves the decoder accepted the envelope and the dispatcher routed the op,
so it is a structural liveness probe.

### Errors

Three error codes are defined (`src/protocol/errno.rs:17`): `EIO = -5` (hardware source failure),
`EINVAL = -22` (short frame, bad length match, or unknown op), and `EMSGSIZE = -90` (a `GET_RANDOM`
over the 4096 ceiling). A malformed envelope that fails to decode at all is answered with `EINVAL`
rather than dropped, so a bad request does not leave the caller waiting on a reply that never comes
(`src/server/runner.rs:41`).

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` initializes the heap and, on success, calls `server::run`;
a heap-init failure exits with code 1 (`src/main.rs:29`). The three top-level modules are `pool` (the
accounting object and the fill path), `protocol` (the wire format), and `server` (the loop, dispatch,
and handlers) (`src/main.rs:22`).

The `Pool` is four relaxed `AtomicU64` counters and nothing else (`src/pool/types.rs:26`):

```
  requests             total GET_RANDOM attempts       (reported as uptime_requests)
  bytes_served         cumulative bytes returned
  last_reseed_request  reseed count (bumped per reseed)
  source_failures      RDRAND give-ups
```

`Pool::new` is a `const fn` that zeroes all four (`src/pool/new.rs:21`). There is no mixed buffer and no
per-caller state; the source is global.

The heart of the capsule is `rdrand_fill` (`src/pool/fill.rs:23`), which draws from the x86 hardware
random generator word by word under a `target_feature = "rdrand"` gate:

```
  #[target_feature(enable = "rdrand")]
  rdrand_fill(out):
      filled = 0
      while filled < out.len():
          word = 0; tries = 0
          while _rdrand64_step(&word) != 1:   // RDRAND can transiently fail
              tries += 1
              if tries >= 32:  return false     // give up on a stalled source
          take = min(8, out.len() - filled)
          out[filled..filled+take] = word.to_le_bytes()[..take]
          filled += take
      return true
```

Each 64-bit `RDRAND` is retried up to 32 times, because the instruction is allowed to fail transiently
when its entropy buffer is momentarily empty; after 32 failures the fill gives up and reports a source
failure (`src/pool/fill.rs:30`). Output is filled eight bytes at a time, the last chunk truncated to
the requested length (`src/pool/fill.rs:34`). There is no software mixing; the bytes are the hardware
generator's output verbatim.

Lifecycle:

1. The kernel spawns the capsule at boot from the microkernel spawn plan (`spawn_entropy` at
   `src/userspace/init/spawn_plan/core.rs:51`), which calls `boot::capsule("ENTROPY", "entropy", ...)`
   against the mirror's `spawn_entropy_capsule` (`src/userspace/init/spawn_plan/core.rs:54`).
2. `spawn_entropy_capsule` (`src/security/entropy_capsule/spawn.rs:40`) decodes the baked trust anchor,
   builds a verified spec with service name `entropy_pool`, port 4100, reply port 4101, the embedded
   ELF/cert/manifest/attestation, and the three requested caps, then spawns it through
   `capsule_spawn::spawn_verified` and records the live pid (`spawn.rs:57`).
3. On success the boot helper prints `[ENTROPY] capsule spawned` and registers the capsule with the
   lifecycle registry (`src/userspace/init/capsule_boot/run.rs:29`); on failure it prints an `[ERROR]`
   line with the mapped `SpawnError` (`src/userspace/init/capsule_boot/run.rs:32`).
4. Inside the capsule, `run` (`src/server/runner.rs:27`) allocates a 4608-byte receive buffer, builds
   a fresh `Pool`, and loops: receive on inbox 0, decode, dispatch, send the reply to
   `KERNEL_REPLY_ENDPOINT`. A non-positive receive length is skipped (`src/server/runner.rs:32`), and a
   decode error is answered with `EINVAL` (`src/server/runner.rs:41`).

## Protocol and IPC

The wire format is authoritative in `src/protocol/types.rs`, and the kernel-side mirror at
`src/security/entropy_capsule/protocol.rs` must match it bit-for-bit; the mirror's constants are
identical (`protocol.rs:25`). The header is 20 bytes, little-endian, packed:

```
  u32 magic         0x4E4F454E "NOEN"      types.rs:26
  u16 version       1                      types.rs:27
  u16 op
  u16 flags
  u16 _reserved
  u32 request_id    echoed, not routed on  types.rs:44
  u32 payload_len   <= MAX_PAYLOAD_BYTES (4096)   types.rs:38
```

The decode rejects a short buffer, a wrong magic, a wrong version, an over-large `payload_len`, or a
frame shorter than header plus declared payload (`src/protocol/decode.rs:30`). It never panics and
never unwraps (`src/protocol/decode.rs:28`). The response reuses the same header and prepends an `i32`
status to its payload (`src/protocol/encode.rs:25`). `request_id` is echoed so the caller can match
replies, but the capsule never routes on it; IPC handles routing through the per-process inbox
(`src/protocol/types.rs:22`).

The one in-tree caller is the kernel-side client under `src/security/entropy_capsule/client/`, driven
by the `CryptoRandom` syscall:

- `handle_crypto_random` (`src/syscall/dispatch/crypto/random.rs:31`) requires `CAP_CRYPTO`, rejects a
  null buffer, a zero length, or a length over 4096, then calls the client's `get_random`
  (`random.rs:39`).
- `client::get_random` (`src/security/entropy_capsule/client/get_random.rs:26`) rejects an over-4096
  request without round-tripping, encodes an `OP_GET_RANDOM` frame with a fresh request id, and does a
  locked round trip through the lifecycle transport to the shared capsule state
  (`client/transport.rs:37`). It maps the status word back to a typed error (`get_random.rs:45`).
- `client::get_stats` (`src/security/entropy_capsule/client/get_stats.rs:31`) first gates on
  `CAP_ENTROPY` through `gate_read` (`src/security/entropy_capsule/capability.rs:23`), then requests the
  32-byte stats blob and decodes the four counters.
- `client::reseed` (`src/security/entropy_capsule/client/reseed.rs:25`) gates on `CAP_ADMIN` through
  `gate_reseed` (`src/security/entropy_capsule/capability.rs:35`), rejects an empty or over-256 seed,
  and sends `OP_RESEED`.

So although the capsule serves `GET_STATS` and `RESEED` without checking a capability itself, the only
path that reaches them enforces `CAP_ENTROPY` for stats and `CAP_ADMIN` for reseed on the kernel side
before the IPC leaves the kernel (`src/security/entropy_capsule/capability.rs:28`,
`src/security/entropy_capsule/capability.rs:40`). The pid used for that check is read from the kernel's
process accounting, never from a caller-supplied payload (`capability.rs:24`).

## Security analysis

The source is the CPU hardware RNG (`RDRAND`), retried up to 32 times per word and checked, with a
clean `EIO` on give-up rather than a silent fallback to a weak source inside the capsule
(`src/pool/fill.rs:28`). Honesty about that source matters: `RDRAND` is a single hardware source with
no software whitening, no health test beyond the retry loop, and no secondary source mixed in at the
capsule. This is weaker than the kernel's own secure RNG, which XORs a software CSPRNG with virtio-rng
and CPU-entropy bytes so its output is no weaker than its strongest source
([kernel secure RNG](../../subsystems/crypto/randomness.md)). The capsule trades that mixing for
simplicity and leans on `RDRAND` alone.

The per-draw cap of 4 KiB bounds the allocation a single request can force
(`src/server/handlers/getrandom.rs:32`), and the fill path re-clamps to the same cap so nothing
downstream can exceed it (`src/pool/fill.rs:43`). The reseed cap of 256 bytes and the strict
length-match check reject a mismatched or oversized reseed frame (`src/server/handlers/reseed.rs:31`,
`reseed.rs:34`). A malformed frame is answered `EINVAL` rather than dropped, so the failure mode is
fail-closed at the protocol layer (`src/server/runner.rs:41`).

The mask is minimal: IPC, Memory, Crypto, and nothing else (`Capsule.mk:17`,
`src/security/entropy_capsule/spawn.rs:54`). No Network and no FileSystem means the bytes it serves
cannot be written to disk or shipped off-box by this capsule, and no hardware/driver/MMIO/IRQ/DMA/PIO
bit means it cannot claim a device even though `RDRAND` is a hardware instruction. Its isolation is
trivial because it holds no per-caller state and no secret; the four counters are non-sensitive, and
they do not persist across a restart (`src/pool/types.rs:26`).

Two boundaries are worth stating precisely. First, the capsule does not itself gate `GET_STATS` or
`RESEED` by capability; the enforcement is on the kernel-client side (`CAP_ENTROPY` for stats,
`CAP_ADMIN` for reseed), so a caller that reached the service on the wire directly, bypassing that
client, would face no in-capsule capability check on those two ops
(`src/security/entropy_capsule/capability.rs:23`). Second, the `CryptoRandom` syscall falls back to the
kernel hardware RNG when the capsule is dead, stale, or fails transport, so a caller still receives real
entropy but not necessarily from this capsule (`src/syscall/dispatch/crypto/random.rs:43`); caller-side
errors such as access-denied, invalid-argument, oversized, and protocol-mismatch are surfaced rather
than masked by the fallback (`random.rs:42`, `random.rs:57`).

Honest gaps: there is no boot-time health check of `RDRAND` availability, so a persistent hardware
failure surfaces as `EIO` at request time rather than at startup (`src/pool/fill.rs:49`); the capsule
has no software CSPRNG and no secondary source of its own, so its output quality is exactly `RDRAND`'s;
the counters are not persisted across a restart; and `RESEED` here does not reseed anything, it only
bumps a counter.

## How to contribute

The source lives at `userland/capsule_entropy/`. The wire format is under `src/protocol/`, the request
loop and handlers under `src/server/`, and the accounting object and fill path under `src/pool/`. Any
change to the wire format must be mirrored bit-for-bit in the kernel-side
`src/security/entropy_capsule/protocol.rs`, which the capsule's own protocol comment calls out as
authoritative (`src/protocol/types.rs:17`).

To add or change an operation:

1. Add the opcode constant in `src/protocol/types.rs:30` and the matching constant in the kernel mirror
   `src/security/entropy_capsule/protocol.rs`.
2. Write the handler as one file under `src/server/handlers/`, exposing `pub fn <op>(pool: &Pool, req:
   Request<'_>) -> Vec<u8>` (or `(req)` if it does not touch the pool, as `healthcheck` does), and
   re-export it from `src/server/handlers/mod.rs:22`.
3. Wire it into the match in `src/server/dispatch.rs:25`.
4. If the kernel needs to call it, add a client under `src/security/entropy_capsule/client/` and
   re-export it from `src/security/entropy_capsule/client/mod.rs:26`, gating it in
   `src/security/entropy_capsule/capability.rs` if it must be capability-checked.

To build and sign the capsule, use the per-slug make targets generated from `Capsule.mk` through
`nonos-mk/capsule.mk` (`nonos-mk/capsule.mk:158`, included at `userland/capsule_entropy/Capsule.mk:20`):

```
  make nonos-mk-entropy              build the capsule ELF
  make nonos-mk-entropy-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-entropy-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-entropy-keys   check the per-capsule signing keys exist
```

For a running kernel that embeds the entropy capsule, `make nonos-mk-entropy-prod` builds the
`microkernel-entropy` profile with the entropy artifacts baked in (`Makefile:915`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` (the decoder is explicit that it never panics and never unwraps, `src/protocol/decode.rs:28`,
and the release profile is `panic = "abort"`, `Cargo.toml:25`); modular files, one unit per file, with
`mod.rs` used only for re-exports (`src/pool/mod.rs`, `src/server/mod.rs`); and the AGPL header at the
top of every source file, matching the header on every existing module (`src/main.rs:1`).

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints `[ENTROPY]
capsule spawned` from the boot log (tag `ENTROPY`, message `capsule spawned`,
`src/userspace/init/capsule_boot/run.rs:29`, format at `src/sys/boot_log/output.rs:33`). An absent line
means the capsule never started, usually a signature, manifest, or capability failure, and the error
path prints an `[ERROR]` line with the mapped `SpawnError` instead
(`src/userspace/init/capsule_boot/run.rs:32`). The mirror's spawn also carries a debug tag
`[ENTROPY-DEBUG] load_elf_executable error:` for an ELF load failure
(`src/security/entropy_capsule/spawn.rs:55`).

Failure modes and where to look:

- A `GET_RANDOM` returns `EIO`. This is the distinctive hardware signature, not a policy error:
  `rdrand_fill` gave up after 32 retries on a stalled source, `pool.fill` returned `-5`, and the handler
  answered `EIO` (`src/pool/fill.rs:49`, `src/server/handlers/getrandom.rs:38`). The `source_failures`
  counter increments on every give-up, so the way to tell a dead source from a busy one is `GET_STATS`:
  a rising `source_failures` count with `EIO` replies is the CPU generator failing. Because there is no
  startup probe, this surfaces at request time, not at boot.
- A `GET_RANDOM` returns `EMSGSIZE`. The request asked for more than 4096 bytes
  (`src/server/handlers/getrandom.rs:32`); the kernel client also rejects an over-4096 request before it
  ever round-trips (`src/security/entropy_capsule/client/get_random.rs:27`).
- Any op returns `EINVAL`. A short or malformed frame, a reseed whose declared length does not match the
  payload, or an unknown op (`src/server/runner.rs:41`, `src/server/handlers/reseed.rs:34`,
  `src/server/dispatch.rs:31`). This is distinct from the `EIO` that means the hardware itself refused.
- Random requests succeed but never touch the capsule. The `CryptoRandom` syscall falls back to the
  kernel hardware RNG when the capsule is dead, stale, or fails transport
  (`src/syscall/dispatch/crypto/random.rs:43`), so a caller can be served while the capsule is down;
  confirm the capsule with the `[ENTROPY] capsule spawned` marker and with `GET_STATS` before assuming
  it handled a given request.

## Source map

```
  userland/capsule_entropy/src/main.rs                 _start: heap_init then server::run
  userland/capsule_entropy/src/server/runner.rs        the loop, decode, EINVAL-on-malformed
  userland/capsule_entropy/src/server/dispatch.rs      op -> handler routing
  userland/capsule_entropy/src/server/handlers/        get_random, get_stats, reseed, healthcheck
  userland/capsule_entropy/src/pool/fill.rs            rdrand_fill + Pool::fill
  userland/capsule_entropy/src/pool/types.rs           Pool + Stats (the four counters)
  userland/capsule_entropy/src/pool/{new,snapshot,record_reseed,encode_stats}.rs   the pool ops
  userland/capsule_entropy/src/protocol/types.rs       the NOEN frame, ops, caps, limits
  userland/capsule_entropy/src/protocol/{decode,encode,errno}.rs   frame decode/encode + error codes
  userland/capsule_entropy/Capsule.mk                  slug, handle, ports, capability mask, mirror
  src/security/entropy_capsule/spawn.rs                the verified kernel-side spawn
  src/security/entropy_capsule/protocol.rs             the bit-for-bit kernel mirror of the wire format
  src/security/entropy_capsule/client/                 the kernel client (get_random/get_stats/reseed)
  src/security/entropy_capsule/capability.rs           CAP_ENTROPY read gate, CAP_ADMIN reseed gate
  src/syscall/dispatch/crypto/random.rs                the CryptoRandom syscall + hardware fallback
  src/userspace/init/spawn_plan/core.rs                spawn_entropy at boot
  src/capabilities/types.rs                            the capability bit values
```

Every reference above is verified against those trees.

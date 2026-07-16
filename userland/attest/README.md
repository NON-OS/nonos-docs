# capsule_attest (full reference)

`capsule_attest` is the system's attestation-information service. It answers questions about the running
system's identity and its stated invariants: a liveness check, a product summary, the boot identity, the
invariant list, and a per-capsule capability-mask table. Its name invites an assumption the code does not
support, and this page is careful about the distinction: the capsule returns authored, human-readable
statements about the system and a boot label, not cryptographic proofs computed at request time. The
statements are true and their cited mechanisms are real kernel machinery, but they are a signed-off audit
manifest, not proof objects. The genuine cryptographic attestation runs in the kernel and is read by the
[boot splash](../boot-splash/README.md) through `mk_attest_status`, not by this capsule. The source is
`userland/capsule_attest/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [What the responses actually are](#what-the-responses-actually-are)
- [The invariants, in full](#the-invariants-in-full)
- [The capsule-mask table](#the-capsule-mask-table)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The capsule is a stateless request-reply service. `_start` initializes the heap and hands control to
`server::run`, which loops forever: receive a framed request on the service port, route it to one of five
read-only handlers, and reply to the attested sender (`src/main.rs:29`, `src/server/runner.rs:27`). There
is no request payload for any operation and no per-caller state; every response is generated server-side
from compile-time constants.

What the capsule reports falls into three groups. A `HEALTHCHECK` op is a liveness ping. The `PROOF_*`
ops return authored data: a product summary (name, tagline, version), the six-entry invariant table
(each a name, a claim, and a mechanism), a boot-identity string with a timestamp, and a per-capsule
capability-mask table. None of these handlers touches the STARK, the zk_kernel, or any signing key. The
`PROOF_*` prefix names the subject of the statements (the system's proofs and properties), not a proof
computed on demand. Documenting the capsule as a system-information and stated-invariant service, rather
than as an attestation engine, is the honest reading of its code.

## Identity

Everything the kernel and the service registry need to name and reach the capsule comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `attest` | `Capsule.mk:1` |
| Service handle | `attest` | `Capsule.mk:2`, `src/userspace/capsule_attest/spawn.rs:29` |
| Domain | `systems.nonos` | `Capsule.mk:3` |
| Namespace | `systems.nonos.attest` | `Capsule.mk:7` |
| Service endpoint | `service:4444:attest` | `Capsule.mk:8`, `spawn.rs:30` |
| Reply endpoint | `reply:4445:endpoint.attest.reply` | `Capsule.mk:9`, `spawn.rs:31`, `spawn.rs:32` |
| Capability mask | `0x19` | `Capsule.mk:13`, `spawn.rs:34` |
| Feature gate | `nonos-capsule-attest` | `Capsule.mk:6` |
| Binary name | `attest` | `Capsule.mk:5`, `Cargo.toml:18` |
| Kernel mirror | `src/userspace/capsule_attest` | `Capsule.mk:14` |

The mask `0x19` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x01  CoreExec   bit()  1    types.rs:56
  0x08  IPC        bit()  8    types.rs:59
  0x10  Memory     bit() 16    types.rs:60
  ----
  0x19  = 1 + 8 + 16
```

The kernel spawn path requests exactly `0x19` and no other bit (`spawn.rs:34`, `spawn.rs:49`). There is
no `Debug` bit (256, `types.rs:64`), which is deliberate: the `Capsule.mk` comment states that a capsule
that emitted `MkDebug` markers would forfeit the credibility of the NO LOGS invariant it reports
(`Capsule.mk:11`). There is no `Crypto` bit (32, `types.rs:61`), which matches its scope: a capsule that
returned cryptographic proofs would need crypto authority, and this one holds none because it returns
authored statements. There is no `FileSystem`, no `Network`, and no hardware capability, so it cannot
read a file, reach the wire, or touch a device.

## Operation reference

Five read-only operations, all with no request payload (`src/protocol/ops.rs:17`):

| Op | Opcode | Reply payload | Handler |
|---|---|---|---|
| `OP_HEALTHCHECK` | `0x0001` | status only (0) | `ops.rs:17`, `server/handlers/health.rs:20` |
| `OP_PROOF_SUMMARY` | `0x0002` | name, tagline, version (length-prefixed) | `ops.rs:18`, `server/handlers/proof_summary.rs:21` |
| `OP_PROOF_INVARIANTS` | `0x0003` | count + six {name, claim, mechanism} tuples | `ops.rs:19`, `server/handlers/proof_invariants.rs:21` |
| `OP_PROOF_BOOT` | `0x0004` | boot ms (u64) + label (length-prefixed) | `ops.rs:20`, `server/handlers/proof_boot.rs:22` |
| `OP_PROOF_CAPSULE_LIST` | `0x0005` | count + {name, mask} entries | `ops.rs:21`, `server/handlers/proof_capsule_list.rs:40` |

`route` validates the header before dispatching and returns a status-only error for anything malformed
(`src/server/handlers/router.rs:24`). The error codes it can return (`src/protocol/errno.rs:17`):

| Status | Value | Meaning | Where |
|---|---|---|---|
| `E_INVAL` | `-22` | reply would not fit the output buffer | `errno.rs:17`, e.g. `proof_boot.rs:25`, `proof_invariants.rs:31`, `proof_capsule_list.rs:43` |
| `E_BAD_OP` | `-38` | opcode is not one of the five | `errno.rs:18`, `router.rs:35` |
| `E_BAD_MAGIC` | `-71` | header magic is not `0x41545354` | `errno.rs:19`, `decode.rs:28` |
| `E_BAD_LEN` | `-90` | buffer shorter than the header, or short payload | `errno.rs:20`, `decode.rs:20`, `decode.rs:35` |
| `E_BAD_VERSION` | `-93` | header version is not 1 | `errno.rs:21`, `decode.rs:31` |

## What the responses actually are

Precisely, the `PROOF_*` operations return pre-authored data, not proof computations.

- `OP_PROOF_SUMMARY` writes three length-prefixed product fields, the name, the tagline, and the version,
  from static constants (`server/handlers/proof_summary.rs:21`). The name is `NONOS`, the tagline is
  `Capability-based RAM-resident microkernel`, and the version comes from `CARGO_PKG_VERSION`
  (`src/state/product.rs:17`).
- `OP_PROOF_INVARIANTS` returns a count followed by a static array of six invariants, each a `name`, a
  `claim`, and a `mechanism`, all authored as constant byte strings (`server/handlers/proof_invariants.rs:21`,
  `src/state/invariants.rs:23`).
- `OP_PROOF_BOOT` returns a timestamp and a fixed label (`server/handlers/proof_boot.rs:22`).
- `OP_PROOF_CAPSULE_LIST` returns a count followed by the authored `{name, mask}` table
  (`server/handlers/proof_capsule_list.rs:40`).

There is no STARK, no zk_kernel, and no signature computed in any of these handlers.

## The invariants, in full

The six invariants (`src/state/invariants.rs:23`) are the heart of the response, and they are worth
reproducing because each `mechanism` names a real kernel component documented elsewhere in this wiki:

```
  NO LOGS
    claim:     no shipped capsule may emit MkDebug or open a serial surface
    mechanism: every shipped Capsule.mk has the Debug bit absent from CAPSULE_REQUIRED_CAPS;
               the kernel rejects MkDebug syscalls outside the mask

  NO TRACES
    claim:     no persistent user identifier or content survives a capsule exit
    mechanism: capsules refuse the FileSystem cap unless granted; the clipboard has idle auto-clear;
               the input_router holds no history

  EPHEMERAL
    claim:     all state is RAM-resident; no on-disk record unless a capsule declares FileSystem
    mechanism: only ramfs + vfs touch disk surfaces; the trust keystore is read-only at boot

  NOT LINUX
    claim:     no POSIX shapes, no errno tables, no fd numbering, no signal model
    mechanism: the Mk* 4-byte ASCII tag syscall ABI; an NCMP-style wire across every capsule;
               a NONOS-native capability taxonomy

  PRIVACY MICROKERNEL
    claim:     every capsule runs CPL=3 with a static capability mask the kernel enforces at every syscall
    mechanism: capsule_spawn::spawn_verified records the caps_bits; syscall dispatch checks the mask
               before every routed handler; the mask is signed in the capsule manifest

  HYBRID-PQ SIGNATURES
    claim:     every binary loaded at runtime is signed Ed25519 + ML-DSA-65 and chains to the anchor
    mechanism: capsule_spawn::spawn_verified rejects any ELF whose nonos_id_cert + manifest do not both
               verify against BAKED_TRUST_ANCHOR_POLICY
```

Each mechanism corresponds to code this wiki documents: the capability-mask enforcement is the
[syscall boundary](../../subsystems/syscall/boundary.md), the hybrid signatures are the
[verified-spawn gate](../../security/capsules-and-trust.md) requiring both Ed25519 and ML-DSA-65, the
RAM-residency is the [zeroization](../../subsystems/memory/zeroization.md) posture, and the `Mk*` ABI is
the [syscall numbers](../../subsystems/syscall/numbers.md). So the invariants are an accurate catalogue of
guarantees, each checkable against the cited code; they are just not proven by this capsule at request
time. The `spawn_verified` path the last two invariants cite is the same one this capsule was loaded
through (`src/userspace/capsule_attest/spawn.rs:52`).

## The capsule-mask table

`OP_PROOF_CAPSULE_LIST` returns a hard-coded table of expected shipped capsules and the capability mask
each is expected to hold (`src/server/handlers/proof_capsule_list.rs:20`). It is not a live process
census and it does not query the kernel for the running caps_bits; it is an authored expectation table
that an auditor can compare against the manifests. The seventeen entries and their declared masks:

```
  ramfs 0x19   vfs 0x19   keyring 0x19   entropy 0x19   crypto 0x19   market 0x19
  clipboard 0x19   attest 0x19   input_router 0x19   wm 0x19
  compositor 0x7819   desktop_shell 0x1819   wallpaper 0x1819   about 0x1819
  calculator 0x1819   terminal 0x1819   driver.virtio_gpu0 0x1F9019
```

The value of the table is exactly the NO LOGS check: every mask can be inspected for the `Debug` bit
(256, `src/capabilities/types.rs:64`), and none of the seventeen carries it, so an auditor can confirm
the NO LOGS posture programmatically without trusting a narrative. The table is authored, so it is a
statement of what the system should ship, not a readout of what is running; the live enforcement is the
kernel's, at every syscall, against the signed manifest.

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` initializes the heap and, on failure, exits with status 1;
on success it enters the server loop, which never returns (`src/main.rs:29`). The three top-level modules
are `protocol` (the wire format), `server` (the loop, router, and handlers), and `state` (the authored
tables) (`src/main.rs:22`).

The server is stateless. `run` allocates two 64 KiB buffers once and loops
(`src/server/runner.rs:27`, `src/protocol/limits.rs:18`):

```
  run():
      loop:
          n = mk_ipc_recv_from(SERVICE_PORT 4444, in_buf, RECV_TIMEOUT_MS 0, &sender_pid)
          if n <= 0 or sender_pid == 0:  mk_yield; continue
          m = route(in_buf[..n], out_buf)
          if m > 0:  mk_ipc_reply(sender_pid, out_buf[..m])
```

The receive timeout is 0, so `mk_ipc_recv_from` does not block indefinitely; an empty receive or a
message from `sender_pid == 0` yields the CPU and retries (`src/server/runner.rs:39`). `route` parses and
validates the header, dispatches to the matching handler, and every handler writes its reply into
`out_buf` and returns the byte count (`src/server/handlers/router.rs:24`).

Lifecycle:

1. The kernel spawns the capsule at boot in the desktop-services fleet, behind the `nonos-capsule-attest`
   feature, as `boot::capsule("ATTEST", "attest", spawn_attest_capsule, shared_state)`
   (`src/userspace/init/spawn_plan/desktop_services.rs:20`, `desktop_services.rs:29`).
2. `spawn_attest_capsule` decodes the baked trust anchor and calls `capsule_spawn::spawn_verified` with
   the embedded ELF, id cert, manifest, and attestation trailer, requesting caps `0x19`
   (`src/userspace/capsule_attest/spawn.rs:36`, `spawn.rs:52`); the embedded bytes come from
   `src/userspace/capsule_attest/embed.rs:18`.
3. On success the boot path prints `[ATTEST] capsule spawned` and registers the capsule with the service
   lifecycle so `attest` resolves through `mk_service_lookup`
   (`src/userspace/init/capsule_boot/run.rs:29`, `src/userspace/capsule_attest/state.rs:21`); on failure
   it prints an `[ERROR]` line with the `SpawnError` (`capsule_boot/run.rs:32`).

## Protocol and IPC

The wire is a 20-byte NCMP-style header followed by a typed payload. The header layout
(`src/protocol/decode.rs:23`, `src/protocol/encode.rs:19`):

```
  offset 0   magic     u32   0x41545354 ("ATST")     header.rs:17
  offset 4   version   u16   1                        header.rs:18
  offset 6   op        u16   the operation opcode
  offset 8   flags     u16   echoed back in the reply
  offset 10  reserved  u16   zero-filled on reply     encode.rs:24
  offset 12  request_id u32  echoed back in the reply
  offset 16  payload_len u32 length of the payload that follows
```

`parse` rejects a buffer shorter than 20 bytes with `E_BAD_LEN`, a wrong magic with `E_BAD_MAGIC`, a
wrong version with `E_BAD_VERSION`, and a payload shorter than `payload_len` with `E_BAD_LEN`
(`src/protocol/decode.rs:19`). Every reply reuses the request's `op`, `flags`, and `request_id`, sets the
reserved field to zero, and carries a 4-byte little-endian status word first, followed by any op payload
(`src/server/respond.rs:19`, `src/protocol/encode.rs:29`). A status of 0 means success; a negative status
is one of the `errno` codes.

The capsule makes no outbound IPC calls. Its only external dependencies are libc syscalls
(`src/main.rs:26`, `src/server/runner.rs:19`, `src/server/handlers/proof_boot.rs:17`):

```
  heap_init          one-time heap setup at _start
  mk_ipc_recv_from   block for a request on port 4444
  mk_ipc_reply       reply to the attested sender pid
  mk_yield           yield when there is nothing to serve
  mk_time_millis     the monotonic boot clock, for OP_PROOF_BOOT only
  mk_exit            exit 1 if heap_init fails
```

`OP_PROOF_BOOT` is the only handler that reads a kernel value at request time: `mk_time_millis` for the
boot timestamp, clamped to 0 if the syscall returns a negative value; the label
`NONOS bootloader (hybrid Ed25519 + ML-DSA-65)` is a fixed byte string
(`src/server/handlers/proof_boot.rs:28`, `proof_boot.rs:39`). This is a boot-identity string plus a
timestamp, not a cryptographic attestation chain.

The real boot attestation, the kernel's measured `zk_verified` and `secure_boot` status, is read through
`mk_attest_status` by the [boot splash](../boot-splash/README.md), not by this capsule
(`userland/capsule_boot_splash/src/main.rs:52`, `userland/libc/src/attest.rs:30`). This capsule never
calls `mk_attest_status`.

## Security analysis

The mask `0x19` grants CoreExec, IPC, and Memory and nothing else (`Capsule.mk:13`,
`src/userspace/capsule_attest/spawn.rs:34`). This is the minimal service set: run user code, speak IPC on
its port, and hold a bounded reply buffer. The mask is itself an illustration of one of the invariants
the capsule reports. It holds no `Debug` bit, so it cannot emit `MkDebug` or open a serial surface, which
is the NO LOGS mechanism the invariant list cites. It holds no `Crypto`, which is the honest match to its
scope: it returns authored statements, not computed proofs, and needs no key material. It holds no
`FileSystem`, no `Network`, and no hardware bit, so it cannot read a file, reach the wire, or touch a
device.

Its isolation is complete because it holds no per-caller state and no secret. Every response is generated
server-side from static constants (`src/state/`), so there is nothing a caller can read out of another
caller, and there is no request payload for a caller to smuggle state through. The parse path validates
the magic, the version, and the declared length before any handler runs, and each handler bounds-checks
its writes against the output buffer and returns `E_INVAL` rather than overrunning
(`src/server/handlers/proof_boot.rs:24`, `proof_invariants.rs:31`, `proof_capsule_list.rs:49`). The
capsule was loaded through the same verified-spawn gate as every other capsule, so its own binary chains
to the baked trust anchor (`spawn.rs:52`).

The honest boundary is the one the whole page draws: the mask cannot make a `PROOF_*` response into a
proof. What this capsule is trusted with is telling the truth about the system, and the security value is
that its authored invariants and its capsule-mask table are *checkable* against the cited code, not that
they are attested at request time. Known gaps, stated as such: the reply carries no cryptographic
signature; the capsule-mask table is authored rather than read from the kernel's live caps_bits; and
`OP_PROOF_BOOT` returns a fixed label rather than a measured boot chain. The genuine cryptographic
machinery lives in the kernel, in the transparent [STARK and Pedersen attestation](../../security/attestation.md)
and the capsule-attestation gate that verifies an enrolled-secret proof at spawn.

## How to contribute

The source lives at `userland/capsule_attest/`. The wire protocol is under `src/protocol/`, the server
loop and per-op handlers under `src/server/`, and the authored tables under `src/state/`. The kernel-side
embed, verified-spawn wiring, and lifecycle state are the mirror at `src/userspace/capsule_attest/`.

To add or change an operation:

1. Add the opcode constant to `src/protocol/ops.rs:17` and re-export it from `src/protocol/mod.rs:29`.
2. Write the handler as one file under `src/server/handlers/`, exposing
   `pub fn run(out: &mut [u8], req: &Request) -> usize` that writes its reply into `out` and returns the
   byte count, bounds-checking every write and returning `respond::status(out, req, E_INVAL)` if the
   reply would not fit (mirror `src/server/handlers/proof_invariants.rs:21`).
3. Wire it into the module list and the match in `src/server/handlers/mod.rs:17` and
   `src/server/handlers/router.rs:29`.
4. If the handler reads new authored data, add it under `src/state/` and re-export it from
   `src/state/mod.rs:20`.

To build and sign the capsule, use the generated per-slug make targets. `nonos-mk/capsule.mk` expands
these from the slug in `Capsule.mk`, which the top-level Makefile includes at `Makefile:682`
(`nonos-mk/capsule.mk:158`):

```
  make nonos-mk-attest              build the capsule ELF
  make nonos-mk-attest-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-attest-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-attest-keys   check the per-capsule signing keys exist
```

The signed artifacts land in `nonos-data/trust/capsules/attest.{nonos_id_cert,manifest,zk_trailer}.bin`,
which the kernel embed includes at build time (`src/userspace/capsule_attest/embed.rs:22`). The capsule is
attested as part of the desktop-services set the boot fleet builds
(`make nonos-mk-all-capsules-attested`, `Makefile:709`), and it is one of the capsules pulled into the
production desktop images (`Makefile:1087`, `Makefile:1116`, `Makefile:1138`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns an error status, never a panic; the release profile is
`panic = "abort"`, `Cargo.toml:25`); modular files, one unit per file, with `mod.rs` used only for
re-exports (`src/server/mod.rs:21`, `src/protocol/mod.rs:24`); and the AGPL header at the top of every
source file, matching the header on every existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints `[ATTEST]
capsule spawned`, written to serial and the on-screen boot log by `boot_log::ok`
(`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line means the
capsule never started, usually a signature, manifest, or capability failure; the error path prints an
`[ERROR]` line with the decoded `SpawnError` instead (`capsule_boot/run.rs:32`,
`src/userspace/init/capsule_boot/error.rs:21`).

Because every response is server-generated and there is no request payload, the failure surface is
narrow:

- A present `[ATTEST]` marker plus a resolving `mk_service_lookup("attest")` is essentially the whole
  health check; a client can also send `OP_HEALTHCHECK`, which replies with a bare success status
  (`src/server/handlers/health.rs:20`).
- A malformed request is rejected before any handler runs. A wrong magic returns `E_BAD_MAGIC`, a wrong
  version returns `E_BAD_VERSION`, and a truncated buffer returns `E_BAD_LEN`
  (`src/protocol/decode.rs:19`); an unknown opcode returns `E_BAD_OP` (`src/server/handlers/router.rs:35`).
- An oversized reply is the only in-handler failure. If a payload would exceed the 64 KiB output buffer,
  the handler returns `E_INVAL` with no partial write (`src/server/handlers/proof_boot.rs:25`,
  `proof_invariants.rs:31`, `proof_capsule_list.rs:49`).

There is no per-request state to corrupt and no secret to leak, so the capsule cannot fail in a way that
exposes another caller's data; the worst case is a status-only error reply.

## Source map

```
  src/main.rs                                   _start -> heap_init -> server::run
  src/protocol/header.rs                        magic 0x41545354, version 1, 20-byte header
  src/protocol/{decode,encode}.rs               parse / response_header + write_status
  src/protocol/ops.rs                           the five opcodes (0x0001..0x0005)
  src/protocol/{errno,limits}.rs                E_* status codes; STATUS_LEN, IPC_PAYLOAD_MAX 64 KiB
  src/server/runner.rs                          the reply-to-sender loop on port 4444
  src/server/respond.rs                         status / with_payload reply builders
  src/server/handlers/router.rs                 header validation + op dispatch
  src/server/handlers/health.rs                 OP_HEALTHCHECK
  src/server/handlers/proof_summary.rs          OP_PROOF_SUMMARY (name/tagline/version)
  src/server/handlers/proof_invariants.rs       OP_PROOF_INVARIANTS (the six invariants)
  src/server/handlers/proof_boot.rs             OP_PROOF_BOOT (mk_time_millis + fixed label)
  src/server/handlers/proof_capsule_list.rs     OP_PROOF_CAPSULE_LIST (the authored mask table)
  src/state/invariants.rs                       the six {name, claim, mechanism} invariants
  src/state/product.rs                          product name, tagline, version constants
  Capsule.mk                                    slug, handle, ports, mask 0x19, kernel mirror
  src/userspace/capsule_attest/                 the kernel-side embed, verified spawn, lifecycle state
  src/userspace/init/spawn_plan/desktop_services.rs   the desktop-services spawn entry
  src/capabilities/types.rs                     the capability-bit definitions the mask decomposes against
  nonos-mk/capsule.mk                           the generated nonos-mk-attest[-sign|-verify] targets
```

Every reference above is verified against those trees.

# capsule_installer

`capsule_installer` is the keystone that turns a request into a running process. It takes either a name
or a full artifact set, marshals the four blobs a capsule image needs, and hands them to the kernel's
verified-load syscall, which runs the entire trust chain before anything spawns. It is the capsule that
loads every other capsule, and it is deliberately the least privileged actor in that transaction: it
holds no keys, verifies nothing itself, and defers every signature, manifest, and attestation check to
the kernel. Its whole job is to move bytes into `mk_capsule_load` and relay the kernel's verdict.

It is spawned as part of the desktop-services fleet at boot under service handle `installer` on service
port 4112 with a reply port on 4113, and its capability mask is `0x19`
(`userland/capsule_installer/Capsule.mk:18`). The source is `userland/capsule_installer/`.

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

The installer is a headless server capsule. `_start` initializes the heap and calls `server::run`, which
loops forever receiving one request, decoding an eight-byte header, dispatching one of four operations,
and sending a reply to the kernel reply endpoint (`src/main.rs:28`, `src/server/runner.rs:26`). It owns
no storage device and no crypto: when a load is by name, it reads the four artifacts from the signed
`/capsules` store through the [vfs](../vfs/README.md) client and passes them straight to the kernel; when a load is
by payload, the caller supplies the bytes and the installer forwards them unchanged. Either way the
security guarantee is the kernel's, not the installer's.

Two roles live in the same capsule. The load role (`LOAD_BY_NAME`, `LOAD_FROM_STORE`) is the verified
spawn path. The install-admission role (`INSTALL`) is the payment gate: for a priced listing it drives
settlement through the [payment](../payment/README.md) capsule and returns a signed receipt hash; for a free
listing it returns an empty receipt without contacting anyone (`src/server/handlers/install.rs:27`). The
installer holds no key material and no index-signature authority; it binds payment to admission and then
lets the kernel bind signatures to execution.

## Identity

Everything the kernel and the service registry need to name and reach the installer comes from its
`Capsule.mk` and its kernel-side spawn record. The two are kept in lockstep: the kernel spawn constants
mirror the `Capsule.mk` fields exactly.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `installer` | `Capsule.mk:7` |
| Service handle | `installer` | `Capsule.mk:8`, `src/userspace/capsule_installer/spawn.rs:28` |
| Namespace | `systems.nonos.installer` | `Capsule.mk:14` |
| Service endpoint | `service:4112:installer` | `Capsule.mk:15`, `spawn.rs:29` |
| Reply endpoint | `reply:4113:endpoint.4294967313` | `Capsule.mk:16`, `spawn.rs:30`, `spawn.rs:31` |
| Capability mask | `0x19` | `Capsule.mk:18`, `spawn.rs:33` |
| Binary name | `installer` | `Capsule.mk:11` |
| Kernel mirror | `src/userspace/capsule_installer` | `Capsule.mk:19` |

The reply inbox name `endpoint.4294967313` is the decimal form of `0x1_0000_0011`, which is the constant
`KERNEL_REPLY_ENDPOINT` the server sends every reply to (`src/protocol/types.rs:22`,
`src/server/runner.rs:40`). Requests arrive on inbox `0`, the service port
(`src/server/runner.rs:31`).

The mask `0x19` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x01  CoreExec   bit()  1   types.rs:56
  0x08  IPC        bit()  8   types.rs:59
  0x10  Memory     bit() 16   types.rs:60
  ----
  0x19  = 1 + 8 + 16
```

The kernel spawn path requests exactly those three capabilities and no others (`spawn.rs:33`,
`spawn.rs:48`), the same minimal set as the [vfs pool](../vfs/README.md). There is no `Crypto` bit (32), so the
installer verifies nothing itself; no `FileSystem` bit (64), so it reads the store over IPC to the vfs
rather than touching a storage surface; and no `Network`, `Driver`, `Mmio`, `Irq`, `Dma`, or `Pio`, so a
bug in it cannot reach hardware or the wire. The authority that matters, the power to spawn a verified
capsule, is not a bit in this mask at all: it is the `mk_capsule_load` syscall, and the kernel gates that
on the trust chain, not on the installer's caps. This is the whole basis of the security analysis below.

The `Capsule.mk` sets `CAPSULE_CARGO_FEATURES := nonos-autorun-install` (`Capsule.mk:13`), a headless
self-verification feature that, at startup and before the request loop, loads the staged `std_proof` and
`rg` packages through the same verified-load syscall so their output lands on serial without needing a
user to open the GUI terminal first (`Cargo.toml:26`, `src/server/selfinstall.rs:30`). It is a build-time
feature of the capsule, not an operation callers can invoke.

## Operation reference

Four operations are defined and dispatched (`src/protocol/types.rs:17`, `src/server/dispatch.rs:26`):

| Op | Opcode | Handler | Reply |
|---|---|---|---|
| `OP_HEALTHCHECK` | 1 | `handlers::health` | `seq | status=0` (liveness) |
| `OP_INSTALL` | 2 | `handlers::install` | receipt hash, or negative errno |
| `OP_LOAD_FROM_STORE` | 3 | `handlers::load_store` | new pid, or the kernel's negative rc |
| `OP_LOAD_BY_NAME` | 4 | `handlers::load_by_name` | new pid, or the kernel's negative rc |

Any other opcode replies `EINVAL` (`-22`) (`src/server/dispatch.rs:31`, `src/protocol/types.rs:26`).
Every request is `seq(4) | op(2) | pad(2) | body`, and every reply is `seq(4) | status(4) | payload`
(`src/protocol/decode.rs:19`, `src/protocol/encode.rs:19`).

### OP_HEALTHCHECK (1)

Liveness probe. Ignores the body and replies `seq | status=0 | (empty)`
(`src/server/handlers/health.rs:21`).

### OP_LOAD_BY_NAME (4)

The elegant path: the caller sends only a name, and the installer reads the four artifacts from the store
itself, so a multi-megabyte ELF never crosses the IPC boundary in one message
(`src/server/handlers/load_by_name.rs:36`). The body layout is:

```
  requested_caps(8) | name_len(1) | name | args
```

The handler bounds-checks the length (at least nine bytes, and `9 + name_len` must not overflow or exceed
the buffer), then requires `valid_name(name)`: non-empty, at most 64 bytes, and every byte in
`[A-Za-z0-9_-]`, so a name can never inject a path escape out of `/capsules`
(`load_by_name.rs:38`, `:43`, `:95`). It then reads
`/capsules/<name>.{elf,nonos_id_cert.bin,manifest.bin,zk_trailer.bin}` through the vfs client, each capped
at 16 MiB (`load_by_name.rs:56`, `:28`), and builds a `CapsuleLoadRequest` carrying the four
pointer/length pairs, the `requested_caps` the caller asked for, and the args blob (`load_by_name.rs:65`).
The four blobs stay owned by the handler's stack frame until `mk_capsule_load` returns, so the kernel
copies from live memory (`load_by_name.rs:78`). On success it replies the new capsule pid as a little
endian `u32` with status `0`; on failure it relays the syscall's negative `rc` as the status
(`load_by_name.rs:81`). Any read failure or a bad name replies `EINVAL`.

The four artifacts are the ELF, the [NONOS-ID certificate](../../security/certificate-schema.md), the
[manifest](../../security/manifest-schema.md), and the ZK attestation trailer
(see [attestation](../../security/attestation.md)).

### OP_LOAD_FROM_STORE (3)

The variant where the caller supplies the artifact bytes directly instead of a name
(`src/server/handlers/load_store.rs:22`). The 28-byte head is:

```
  requested_caps(8) | elf_len(4) | cert_len(4) | manifest_len(4) | trailer_len(4) | args_len(4)
```

followed by the ELF, cert, manifest, trailer, and args blobs in that order. Every offset is folded with
`checked_add`, and the total must equal the payload length exactly, so a malformed length field or a
truncated body replies `EINVAL` rather than reading out of bounds
(`load_store.rs:33`, `:48`). The blobs are sliced in place and passed to the same `mk_capsule_load`; the
reply is identical to the by-name path: the new pid on success, or the kernel's negative `rc`
(`load_store.rs:64`). The by-name path is preferred in practice because it avoids shipping the ELF over
IPC.

For both load ops the `requested_caps` field is a request, not a grant. It is the upper bound the caller
is willing to see granted, and the kernel bounds it further: the verified manifest and the certificate
ceiling are the real limits, and a capsule is never handed a capability it did not declare
(`src/kernel_core/process_spawn/capsule_spawn/from_vfs/load.rs:34`, and see the
[security analysis](#security-analysis)).

### OP_INSTALL (2)

The payment-admission path (`src/server/handlers/install.rs:27`). The body is a fixed 125 bytes:

```
  owner_pid(4) | wallet_id(4) | price_kind(1) | capsule_id(32) | publisher(20) | amount(32) | receipt_type(32)
```

decoded into `InstallReq` (`src/server/fields.rs:17`, `install.rs:33`). If the length is wrong it replies
`EINVAL` (`install.rs:29`). If `price_kind == PRICE_KIND_FREE` (0) it returns a 32-byte zero receipt with
status `0` and contacts no one (`install.rs:42`, `src/server/consts.rs:19`). Otherwise it resolves the
`payment` service by name; if that lookup fails it replies `EAGAIN` (`-11`)
(`install.rs:45`, `src/server/discover.rs:21`, `src/protocol/types.rs:27`). It then calls the payment
capsule's `OP_PAY` (2) with the owner pid, wallet id, capsule id, publisher, amount, and receipt type,
and on a `status == 0` reply returns the 32-byte signed struct hash; a non-zero payment status is
relayed as the errno (`src/server/pay_call.rs:25`, `src/server/consts.rs:18`). The installer holds no
receipt state beyond the in-flight call.

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` initializes the heap and calls `server::run`, exiting with
code 1 if the heap fails to initialize (`src/main.rs:28`). The two top-level modules are `protocol` (the
wire codec) and `server` (the loop, dispatch, discovery, and handlers) (`src/main.rs:22`).

The `protocol` module is a thin codec: `decode_request` splits a frame into `seq`, `op`, and a payload
slice, and returns `None` on a short frame; `encode_response` prepends `seq` and `status` to a payload
(`src/protocol/decode.rs:19`, `src/protocol/encode.rs:19`). The `server` module holds the runner, the
dispatch table, the payment-service discovery helper, the `InstallReq` field decoder, the `addr20`/`word32`
byte extractors, and the four handlers (`src/server/mod.rs:17`). The `selfinstall` module is compiled only
under `nonos-autorun-install` (`src/server/mod.rs:25`).

Lifecycle:

1. The kernel spawns the capsule at boot through the desktop-services fleet plan, which calls
   `spawn_installer` (gated on the `nonos-capsule-installer` feature) and runs
   `boot::capsule("INSTALLER", "installer", ...)`
   (`src/userspace/init/spawn_plan/desktop_services.rs:21`, `:35`, `:37`). That verifies the embedded ELF,
   cert, manifest, and attestation against the baked trust anchor and registers `installer` on port 4112
   (`src/userspace/capsule_installer/spawn.rs:35`).
2. On success the boot helper logs `[INSTALLER] capsule spawned` and registers the capsule with the
   lifecycle table; on failure it logs an `[ERROR]` line carrying the `SpawnError`
   (`src/userspace/init/capsule_boot/run.rs:29`, `:32`, `src/sys/boot_log/output.rs:33`, `:49`).
3. Under `nonos-autorun-install`, `server::run` first calls `selfinstall::run`, which waits for the vfs
   store to answer and then loads `std_proof` and `rg` through the verified-load syscall
   (`src/server/runner.rs:27`, `src/server/selfinstall.rs:30`).
4. The request loop receives up to 8 MiB per message on inbox `0` (a capsule image is large), decodes the
   header, dispatches, and replies to `KERNEL_REPLY_ENDPOINT`; a non-positive receive or an undecodable
   frame is skipped (`src/server/runner.rs:24`, `:30`).

## Protocol and IPC

Inbound wire, service `installer` on port 4112: `seq(4) | op(2) | pad(2) | body`, reply
`seq(4) | status(4) | payload` (`src/protocol/decode.rs:19`, `src/protocol/encode.rs:19`). The reply goes
to the kernel reply endpoint `0x1_0000_0011` (`src/protocol/types.rs:22`, `src/server/runner.rs:40`).

The single kernel call that gives the installer its purpose is `mk_capsule_load`, invoked by both load
ops and by the self-install path (`src/server/handlers/load_by_name.rs:80`,
`src/server/handlers/load_store.rs:64`, `src/server/selfinstall.rs:72`). The libc shim passes a pointer to
a `CapsuleLoadRequest` through syscall `MCLD` (`userland/libc/src/capsule_load.rs:46`,
`userland/libc/src/syscall/numbers/core.rs:19`). The request struct mirrors the kernel layout exactly:
the four artifact pointer/length pairs, `requested_caps`, and the args pointer/length
(`userland/libc/src/capsule_load.rs:28`, `src/syscall/microkernel/capsule_load/request.rs`). The syscall
returns the new pid, or a stable negative errno: `-13` rejected by verification, `-14` fault, `-22`
invalid request (`userland/libc/src/capsule_load.rs:44`).

Kernel side, `SYS_CAPSULE_LOAD` (also `MCLD`) dispatches to `sys_capsule_load`, which validates and copies
the user struct, reads the four blobs and the args out of user memory, and calls `load_capsule_from_vfs`
(`src/syscall/microkernel/dispatch/process.rs:35`, `src/syscall/microkernel/capsule_load/handle.rs:27`,
`src/syscall/microkernel/numbers.rs:33`). Loader failures map to negative errnos: a malformed manifest is
`EINVAL`, and a rejected signature, manifest, or attestation, or a trust-anchor decode failure, is
`EACCES` (`src/syscall/microkernel/capsule_load/errno.rs:23`).

The other outbound call is to the [payment](../payment/README.md) service. The installer resolves it by name with
`mk_service_lookup` (`src/server/discover.rs:24`) and drives settlement with `mk_ipc_call` to `OP_PAY`
(`src/server/pay_call.rs:37`). The reply is `seq | status | struct_hash(32)`; a reply shorter than 40
bytes, or a non-zero status, is surfaced as an error (`pay_call.rs:38`).

## Security analysis

This is a trust-critical capsule, and its safety argument is the inverse of what the name suggests: the
capsule that loads every other capsule is the least privileged one in the loading transaction.

**All verification is the kernel's.** The installer holds no keys and performs no signature checks. It
cannot be tricked into loading an unsigned or tampered image, because `mk_capsule_load` runs the full
[verified-spawn](../../security/capsules-and-trust.md) pipeline on the artifacts. The kernel takes the
service name, endpoints, and target triple from the capsule's own signed manifest, not from the caller, so
a loaded capsule registers exactly what it declares and cannot be misnamed or misrouted
(`src/kernel_core/process_spawn/capsule_spawn/from_vfs/load.rs:36`). It decodes and verifies the
NONOS-ID certificate against the baked [trust anchor](../../security/trust-anchor.md), verifies the
manifest publisher signatures, and requires both an Ed25519 and an ML-DSA-65 signature: the production
policy lists both algorithms as required and every listed algorithm must verify
(`src/security/nonos_id_cert/policy.rs:31`, `src/security/capsule_manifest/verify/mod.rs:53`,
`src/security/capsule_manifest/verify/dispatch.rs:47`). Only an image that passes all of this, including
the [attestation gate](../../security/attestation.md), is spawned; a rejected image surfaces as a negative
errno at the syscall, not as anything the installer decided.

**Requested caps are bounded, not granted on request.** The `requested_caps` field the caller sends is an
upper bound for the optional caps, identical in meaning to a baked spawn site's request; the verified
manifest still decides what is actually granted
(`src/kernel_core/process_spawn/capsule_spawn/from_vfs/load.rs:34`). Two checks enforce this. First the
manifest's own declared caps (required union optional) must not exceed the certificate ceiling, or the
load fails with `manifest:caps_ceiling` (`src/security/capsule_manifest/verify/caps.rs:20`,
`from_vfs/load.rs:90`). Then the grant check requires the requested caps to be a subset of what the
manifest declares, and the caps actually installed are `required_caps | (optional_caps & requested)`, so
asking for `u64::MAX` (as the self-install and the terminal's install client do) can only ever select
within the manifest's own optional set, never widen it (`src/security/capsule_manifest/verify/caps.rs:31`).

**The installer itself is trusted with almost nothing.** Its mask is `CoreExec | IPC | Memory` and no
more (`Capsule.mk:18`, `src/userspace/capsule_installer/spawn.rs:33`). It has no crypto, so it verifies
nothing; no filesystem cap, so it reaches the store only by asking the vfs; no hardware, driver, or DMA
authority. The by-name path validates the name to `[A-Za-z0-9_-]`, at most 64 bytes, non-empty, before
building any store path, so it cannot escape `/capsules`
(`src/server/handlers/load_by_name.rs:95`). The by-payload path folds every artifact offset with
`checked_add` and requires the total to match the message length, so a crafted length cannot cause an
out-of-bounds read (`src/server/handlers/load_store.rs:33`). The artifact blobs are stack-owned across the
syscall, so there is no use-after-free of the bytes the kernel copies
(`src/server/handlers/load_by_name.rs:78`).

**Honest gaps.** The installer has no caller attestation: its inbox is a public port, so any capsule that
can reach it can send a load request. But because the kernel's trust chain gates every spawn, a load only
succeeds for a properly signed, in-policy image, so the exposure is a denial-of-service surface rather than
an authenticity one. The installer holds no persistent state beyond an in-flight payment call, so there is
nothing to leak across the boot. There is no reject-all development stub here; the `offline-verify` stub
lives on the sibling [market](../market/README.md) capsule, which verifies content itself, whereas the installer
defers verification entirely.

## How to contribute

The source lives at `userland/capsule_installer/`. The wire codec is under `src/protocol/`, and the
server (loop, dispatch, discovery, handlers) is under `src/server/`, one unit per file. To add or change
an operation:

1. Define the opcode in `src/protocol/types.rs:17` and re-export it from `src/protocol/mod.rs:23`.
2. Write the handler as one file under `src/server/handlers/`, exposing
   `pub fn <name>(req: Request<'_>) -> Vec<u8>` that returns an `encode_response`, and re-export it from
   `src/server/handlers/mod.rs:17`.
3. Wire it into the match in `src/server/dispatch.rs:26`. Reply to the unknown-op arm with `EINVAL` for
   anything you do not handle.

To build and sign the capsule, use the generated per-slug make targets. The `Capsule.mk` includes
`nonos-mk/capsule.mk`, which defines them for the `installer` slug
(`userland/capsule_installer/Capsule.mk:21`, `nonos-mk/capsule.mk:158`):

```
  make nonos-mk-installer              build the capsule ELF
  make nonos-mk-installer-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-installer-verify       verify the signed manifest against the trust anchor
  make nonos-mk-check-installer-keys   assert the per-capsule signing keys exist
```

These map to `nonos-mk/capsule.mk:182` (build), `:261` (sign), `:263` (verify), and `:184` (check keys).
The top-level `Makefile` includes the installer's `Capsule.mk` and folds its verify and artifacts into the
image build (`Makefile:654`, `Makefile:724`, `Makefile:1077`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns errors as an `encode_response` with a negative status,
never a panic; the release profile is `panic = "abort"`, `Cargo.toml:33`); modular files, one unit per
file, with `mod.rs` used only for re-exports; and the AGPL header at the top of every source file, matching
the header on every existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[INSTALLER] capsule spawned` (tag `INSTALLER`, message `capsule spawned`) from the boot log
(`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line means the
capsule never started, usually a signature, manifest, or capability failure; the error path prints an
`[ERROR]` line carrying the `SpawnError` instead (`src/userspace/init/capsule_boot/run.rs:32`). A present
marker means `mk_service_lookup("installer")` resolves, so the market, the terminal, and the desktop can
request loads; an absent one means nothing can install.

Failure modes and where to look:

- Install denied at the syscall. The installer's own replies are thin: a load returns the new pid on
  success or the negative `rc` from `mk_capsule_load` on failure
  (`src/server/handlers/load_by_name.rs:81`). The real verdict is the kernel's, and the runtime-load path
  prints it: `[RUNTIME-LOAD] verify start name=...`, then `[RUNTIME-LOAD] spawned name=... pid=...` on
  success or `[RUNTIME-LOAD] FAILED name=... reason=...` on failure, where `reason` names the exact check
  that rejected it (`id_cert`, `manifest:pub_sig`, `manifest:caps_ceiling`, `attestation`, and so on)
  (`src/kernel_core/process_spawn/capsule_spawn/from_vfs/load.rs:68`, `:105`). A `manifest:caps_ceiling`
  or `manifest:grant` reason is a capability request outside what the manifest declares, not an installer
  bug.
- Capsule not found. A by-name load of a name with no artifacts in `/capsules` fails the four-way read and
  replies `EINVAL` before the syscall (`src/server/handlers/load_by_name.rs:56`). A name that is empty,
  over 64 bytes, or contains a byte outside `[A-Za-z0-9_-]` is rejected by `valid_name` with `EINVAL`,
  also before any IPC (`load_by_name.rs:95`).
- Paid install stalls. `OP_INSTALL` replies `EAGAIN` (`-11`) when the payment service is not reachable by
  name, and relays a non-zero payment status verbatim (`src/server/handlers/install.rs:45`,
  `src/server/pay_call.rs:42`).
- Self-install path. Under `nonos-autorun-install`, `selfinstall` logs `[SELF-INSTALL] waiting for vfs
  store`, then `[SELF-INSTALL] loading <name>`, and `[SELF-INSTALL] loaded + spawned ok` or
  `[SELF-INSTALL] load REJECTED`; a store that never answers logs `[SELF-INSTALL] vfs store never ready`
  (`src/server/selfinstall.rs:31`, `:46`, `:74`, `:37`).

## Source map

```
  src/main.rs                              _start -> heap_init -> server::run
  src/protocol/types.rs                    the ops, KERNEL_REPLY_ENDPOINT, errnos, Request
  src/protocol/{decode,encode}.rs          the frame codec (seq | op | pad | body)
  src/server/runner.rs                     the receive/dispatch/reply loop
  src/server/dispatch.rs                   the op match table
  src/server/handlers/health.rs            OP_HEALTHCHECK
  src/server/handlers/load_by_name.rs      OP_LOAD_BY_NAME: read /capsules artifacts + load
  src/server/handlers/load_store.rs        OP_LOAD_FROM_STORE: load from IPC-supplied blobs
  src/server/handlers/install.rs           OP_INSTALL: the payment-admission path
  src/server/{discover,pay_call,consts}.rs the payment service lookup and settlement call
  src/server/{fields,addr20,word32}.rs     the InstallReq decoder and byte extractors
  src/server/selfinstall.rs                the nonos-autorun-install boot self-verification
  Capsule.mk                               slug, handle, ports, capability mask, kernel mirror
  userland/libc/src/capsule_load.rs        the mk_capsule_load shim and CapsuleLoadRequest
  src/userspace/capsule_installer/         the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/desktop_services.rs   the desktop-fleet spawn entry
  src/syscall/microkernel/capsule_load/    the kernel syscall handler
  src/kernel_core/process_spawn/capsule_spawn/from_vfs/load.rs   the verified-load path
  src/security/capsule_manifest/verify/    the signature, ceiling, and grant checks
  nonos-mk/capsule.mk                      the generated nonos-mk-installer[-sign|-verify] targets
```

Every reference above is verified against those trees.

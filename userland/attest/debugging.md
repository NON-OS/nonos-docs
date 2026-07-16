# Debugging

What to check when the attest capsule does not answer, and why its failure surface is narrow. Back to the
[hub](README.md).

## Confirm it ran

On a successful boot the kernel prints `[ATTEST] capsule spawned`, written to serial and the on-screen boot
log by `boot_log::ok` (`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An
absent line means the capsule never started, usually a signature, manifest, or capability failure; the
error path prints an `[ERROR]` line with the decoded `SpawnError` instead
(`capsule_boot/run.rs:32`, `src/userspace/init/capsule_boot/error.rs`). The capsule is behind the
`nonos-capsule-attest` feature, so a build without that feature has no `[ATTEST]` line by design
(`src/userspace/init/spawn_plan/desktop_services.rs:26`).

Inside the capsule, `_start` exits with status 1 if `heap_init` fails and otherwise never returns from
`server::run` (`src/main.rs:29`). There is no later marker to watch for; the spawn line plus a resolving
service lookup is the whole liveness signal.

## Confirm it answers

A present `[ATTEST]` marker plus a resolving `mk_service_lookup("attest")` is essentially the whole health
check. A client can also send `OP_HEALTHCHECK` (`0x0001`), which replies with a bare success status and
reads no state, so a status-0 reply on that opcode proves the loop is serving
(`src/server/handlers/health.rs:20`).

## The failure modes

Because every response is server-generated and no operation carries a request payload, the failure surface
is narrow.

- A malformed request is rejected before any handler runs. A wrong magic returns `E_BAD_MAGIC` (`-71`), a
  wrong version returns `E_BAD_VERSION` (`-93`), and a buffer shorter than the 20-byte header or shorter
  than its declared payload returns `E_BAD_LEN` (`-90`) (`src/protocol/decode.rs:19`). An opcode outside
  the five returns `E_BAD_OP` (`-38`) (`src/server/handlers/router.rs:35`).
- An oversized reply is the only in-handler failure. If a payload would exceed the 64 KiB output buffer,
  the handler returns `E_INVAL` (`-22`) with no partial write
  (`src/server/handlers/proof_boot.rs:25`, `proof_invariants.rs:31`, `proof_capsule_list.rs:49`). For the
  fixed-size tables this capsule serves, this cannot happen in practice; the check is defensive.
- An empty receive or a message from `sender_pid == 0` is not an error. The loop yields and retries, so a
  client that never gets a reply is usually not reaching port 4444 at all, not hitting a capsule fault
  (`src/server/runner.rs:39`).

The status word is the whole diagnostic. It is a 4-byte little-endian value immediately after the 20-byte
header; 0 is success and the negative values above map to the error codes in
[protocol.md](protocol.md). Decoding it tells you exactly which validation step rejected the request.

## What cannot go wrong

There is no per-request state to corrupt and no secret to leak, so the capsule cannot fail in a way that
exposes another caller's data; the worst case is a status-only error reply. If a `PROOF_` reply looks
wrong, the fault is almost always in the request framing or in the client's decode of the length-prefixed
fields, not in capsule state, because the payload is a fixed function of compile-time constants and, for
`OP_PROOF_BOOT` only, the boot clock.

## Source map

```
  src/main.rs                              exit 1 on heap_init failure; otherwise server::run
  src/server/runner.rs                     the recv-reply loop; yield on empty/zero sender
  src/server/handlers/router.rs            E_BAD_OP for unknown opcodes
  src/server/handlers/health.rs            OP_HEALTHCHECK, the client-side liveness probe
  src/protocol/decode.rs                   E_BAD_MAGIC / E_BAD_VERSION / E_BAD_LEN
  src/userspace/init/capsule_boot/run.rs   the [ATTEST] / [ERROR] boot markers
  src/sys/boot_log/output.rs               boot_log::ok, where the marker is written
```

Every reference above is verified against `userland/capsule_attest/` and the cited kernel trees.
</content>

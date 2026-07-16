# Debugging capsule_installer

This page lists the log markers the installer and its load path emit, and the concrete failure modes with
where to look for each. For what the installer does see the [README](README.md), the
[operations](operations.md), and the [verified-load](verified-load.md) pages in this folder.

## Log markers

The first thing to confirm is that the capsule ran. On a successful boot the kernel logs
`[INSTALLER] capsule spawned` from the capsule boot path: the `Ok` arm calls `boot_log::ok("INSTALLER",
"capsule spawned")`, which prints `[<tag>] <msg>` (`src/userspace/init/capsule_boot/run.rs:29`,
`src/sys/boot_log/output.rs:33`). If that line is absent the capsule never started, and the `Err` arm
logged an `[ERROR]` line carrying the `SpawnError` instead
(`src/userspace/init/capsule_boot/run.rs:32`, `src/sys/boot_log/output.rs:49`), which is the usual
signature, manifest, or capability failure at the installer's own spawn. A present marker means
`mk_service_lookup("installer")` resolves, so the market, the terminal, and the desktop can request loads;
an absent one means nothing can install.

The load path prints the kernel's verdict, not the installer's. Every load, whether by name, by payload,
or from self-install, drives `mk_capsule_load`, and the kernel logs `[RUNTIME-LOAD] verify start name=...`,
then either `[RUNTIME-LOAD] spawned name=... pid=...` on success or `[RUNTIME-LOAD] FAILED name=...
reason=...` on failure, where `reason` names the exact check that rejected it
(`src/kernel_core/process_spawn/capsule_spawn/from_vfs/load.rs:68`, `:73`, `:105`).

## Failure modes

### Install denied at the syscall

The installer's own replies are thin: a load returns the new pid on success or the negative `rc` from
`mk_capsule_load` on failure (`src/server/handlers/load_by_name.rs:81`,
`src/server/handlers/load_store.rs:64`). The real verdict is the kernel's, on the `[RUNTIME-LOAD] FAILED`
line. The `reason` field is the fast diagnostic:

- `id_cert` is a rejected NONOS-ID certificate (`from_vfs/load.rs:84`).
- `manifest:pub_sig` is a publisher signature that did not verify; `manifest:pub_revoked` and
  `manifest:pub_policy` are a revoked or unknown publisher key (`from_vfs/load.rs:87`, `:88`, `:89`).
- `manifest:caps_ceiling` is a manifest whose declared caps exceed the certificate ceiling, and
  `manifest:grant` is a requested cap outside what the manifest declares
  (`from_vfs/load.rs:90`, `:95`, `src/security/capsule_manifest/verify/caps.rs:20`, `:31`). Neither is an
  installer bug: they are a capability request outside policy.
- `attestation` is a failed [attestation](../../security/attestation.md) gate (`from_vfs/load.rs:98`).

At the syscall boundary the errno is coarser: a malformed manifest is `EINVAL` and any rejected signature,
manifest, attestation, or trust-anchor decode is `EACCES`
(`src/syscall/microkernel/capsule_load/errno.rs:23`). The libc shim documents the caller-visible codes:
`-13` rejected by verification, `-14` fault, `-22` invalid request
(`userland/libc/src/capsule_load.rs:44`).

### Capsule not found

A by-name load of a name with no artifacts in `/capsules` fails the four-way read and replies `EINVAL`
before the syscall runs (`src/server/handlers/load_by_name.rs:56`, `:62`). All four of
`.elf`, `.nonos_id_cert.bin`, `.manifest.bin`, and `.zk_trailer.bin` must read, so a partial artifact set
looks identical to a missing capsule at this layer.

A name that is empty, over 64 bytes, or contains a byte outside `[A-Za-z0-9_-]` is rejected by `valid_name`
with `EINVAL`, also before any store read or IPC (`src/server/handlers/load_by_name.rs:51`, `:95`). If a
name that should be valid is bounced, check for a stray path separator or a non-ascii byte; the rule
refuses `/` and `.` deliberately so a name cannot escape `/capsules`.

### Paid install stalls

`OP_INSTALL` replies `EAGAIN` (`-11`) when the `payment` service cannot be resolved by name
(`src/server/handlers/install.rs:45`, `src/server/discover.rs:26`). A non-zero payment status, or a
payment reply shorter than 40 bytes, is relayed verbatim as the errno
(`src/server/pay_call.rs:38`, `:42`). A free listing (`price_kind == 0`) never contacts payment and always
returns a zero receipt, so a stall on a free install is not a payment problem
(`src/server/handlers/install.rs:42`).

### Self-install path

Under `nonos-autorun-install`, `selfinstall` logs `[SELF-INSTALL] waiting for vfs store`, then
`[SELF-INSTALL] loading <name>`, and `[SELF-INSTALL] loaded + spawned ok` or `[SELF-INSTALL] load
REJECTED`; a store that never answers within the retry budget logs `[SELF-INSTALL] vfs store never ready`
and a failed artifact read logs `[SELF-INSTALL] artifact read failed`
(`src/server/selfinstall.rs:31`, `:37`, `:47`, `:56`, `:74`, `:76`). A `REJECTED` here is the same
kernel verdict as a normal load; look for the paired `[RUNTIME-LOAD] FAILED` line for the reason.

## Source map

```
  src/userspace/init/capsule_boot/run.rs        [INSTALLER] capsule spawned / [ERROR] path
  src/sys/boot_log/output.rs                    the ok/error line format
  src/kernel_core/process_spawn/capsule_spawn/from_vfs/load.rs   [RUNTIME-LOAD] markers and reason strings
  src/security/capsule_manifest/verify/caps.rs  the ceiling and grant checks behind the reasons
  src/syscall/microkernel/capsule_load/errno.rs the EINVAL / EACCES mapping
  userland/libc/src/capsule_load.rs             the caller-visible errno documentation
  userland/capsule_installer/src/server/handlers/load_by_name.rs  four-way read, valid_name, EINVAL
  userland/capsule_installer/src/server/handlers/load_store.rs    the by-payload reply
  userland/capsule_installer/src/server/handlers/install.rs       EAGAIN and payment status relay
  userland/capsule_installer/src/server/{discover,pay_call}.rs    payment lookup and settlement
  userland/capsule_installer/src/server/selfinstall.rs            the [SELF-INSTALL] markers
```

Every reference above is verified against those trees.

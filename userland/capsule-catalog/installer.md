# capsule_installer

`capsule_installer` loads capsules into the running system. It takes either a payload or a name to read
from the signed `/capsules` store, marshals the four artifacts an image needs, and hands them to the
kernel's capsule-load syscall, which runs the full trust chain before anything spawns. The installer is
stateless and deliberately defers all verification to the kernel: its job is to move bytes, and the
security guarantee is the kernel's. Service `installer` on port 4112, reply endpoint `0x1_0000_0011`,
capability mask `0x19`. The source is `userland/capsule_installer/`.

## Contents

- [The server loop](#the-server-loop)
- [The operations](#the-operations)
- [Loading by name](#loading-by-name)
- [Loading from a payload](#loading-from-a-payload)
- [Where trust lives](#where-trust-lives)
- [The paid-install path](#the-paid-install-path)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop

`main.rs:28` initializes the heap and calls `server::run` (`src/server/runner.rs:24`) with an 8 MiB
message buffer, because a capsule image is large, decoding an eight-byte-header frame and dispatching four
operations:

```
  run():
      loop:
          n = mk_ipc_recv(inbox 0, buf)          // up to 8 MiB
          req = decode_request(buf[..n])          // u32 seq, u16 op, u16 pad
          resp = dispatch(req)
          mk_ipc_send(KERNEL_REPLY_ENDPOINT, resp)
```

## The operations

Four operations (`src/protocol/types.rs:17`):

```
  1  HEALTHCHECK    2  INSTALL    3  LOAD_FROM_STORE    4  LOAD_BY_NAME
```

## Loading by name

`load_by_name` (`src/server/handlers/load_by_name.rs:36`) is the elegant path: the caller sends only a
name, and the installer reads the four artifacts from the signed store itself, so a multi-megabyte ELF
never crosses the IPC boundary in one message:

```
  load_by_name(req):
      payload = requested_caps(8) | name_len(1) | name | args
      require valid_name(name)                    // non-empty, <= 64, [A-Za-z0-9_-] only
      pid = mk_getpid()
      elf      = read /capsules/<name>.elf                 (<= 16 MiB)
      cert     = read /capsules/<name>.nonos_id_cert.bin
      manifest = read /capsules/<name>.manifest.bin
      trailer  = read /capsules/<name>.zk_trailer.bin
      load = CapsuleLoadRequest { elf_ptr/len, cert_ptr/len, manifest_ptr/len,
                                  trailer_ptr/len, requested_caps, args_ptr/len }
      rc = mk_capsule_load(&load)                  // the kernel runs the trust chain here
      return rc >= 0 ? capsule_pid : rc
```

The four artifacts, the ELF, the [NØNOS-ID certificate](../../security/certificate-schema.md), the
[manifest](../../security/manifest-schema.md), and the ZK trailer, are read from
`/capsules/<name>.{elf,nonos_id_cert.bin,manifest.bin,zk_trailer.bin}` through the [vfs](vfs.md), each
capped at 16 MiB. The name is strictly validated (alphanumeric, underscore, hyphen, at most 64 bytes) so
a name cannot inject a path escape. The blobs are owned by the handler's stack frame until
`mk_capsule_load` returns, so the kernel copies from live memory.

## Loading from a payload

`LOAD_FROM_STORE` (`src/server/handlers/load_store.rs`) is the variant where the caller supplies the
artifact bytes directly in the request (the requested capabilities and the four blobs with
overflow-checked lengths), which the installer passes to the same `mk_capsule_load`. This is used when the
image is not already in `/capsules`; the by-name path is preferred because it avoids shipping the ELF over
IPC.

## Where trust lives

The installer verifies nothing itself. `mk_capsule_load` is a kernel syscall, and the kernel runs the
full [verified-spawn](../../security/capsules-and-trust.md) pipeline on the artifacts: it decodes and
verifies the NØNOS-ID certificate against the baked [trust anchor](../../security/trust-anchor.md),
verifies the manifest against the publisher keys, requires **both** an Ed25519 and an ML-DSA-65 signature,
checks the requested capabilities against what the manifest and policy allow, and runs the
[attestation gate](../../security/attestation.md). Only an image that passes all of this is loaded. So the
installer is a loader, and a malicious or corrupt image is rejected by the kernel, not by the installer;
this is the honest framing of its trust boundary.

## The paid-install path

`INSTALL` (`src/server/handlers/install.rs:27`) is the paid path: for a priced capsule it calls the
[payment](payment.md) service with a receipt input and returns the signed receipt hash; for a free one
(`PRICE_KIND_FREE`) it returns an empty receipt immediately. It returns `EAGAIN` if the payment service is
not reachable.

## Security analysis

- **All verification is the kernel's.** The installer holds no keys and performs no signature checks; it
  cannot be tricked into loading an unsigned image because `mk_capsule_load` enforces the trust chain.
- **The name is validated** before it is used to build a store path, so a by-name load cannot escape
  `/capsules`.
- **The blobs are stack-owned** across the syscall, so there is no use-after-free of the artifacts.

The capability mask is `0x19` (`Capsule.mk`, `CAPSULE_REQUIRED_CAPS`), decoding to CoreExec (1), IPC (8),
and Memory (16), the same minimal set as the [vfs pool](vfs.md). This is the striking part of the
installer's design: the capsule that loads *every other capsule* holds no elevated capability of its own.
It has no Crypto, so it verifies nothing itself; the trust chain lives entirely behind `mk_capsule_load` in
the kernel. It has no FileSystem cap, so it reads the four `/capsules` artifacts over IPC to the
[vfs](vfs.md) rather than touching a storage surface directly. It has no Driver, Mmio, Irq, Dma, Pio, or
Network, so a bug in the installer cannot reach hardware or the wire. The privilege that matters, the
authority to spawn a verified capsule, is not a bit in this mask at all: it is the `mk_capsule_load`
syscall, and the kernel gates *that* on the trust chain, not on the installer's caps. So the isolation
argument is inverted from what the name suggests: the installer is deliberately unprivileged, and its power
to install comes only from a syscall whose every success is a signed image. The honest boundary is that the
inbox is a public port with no caller attestation, as the [gaps](#honest-gaps) state, so the exposure is
denial-of-service, not authenticity.

## Debugging

The service is `installer` on port 4112 (`Capsule.mk`, `service:4112:installer`), reply endpoint
`0x1_0000_0011`, and it is spawned in the desktop-services fleet at `spawn_plan/desktop_services.rs:37`
(behind the `nonos-capsule-installer` feature) as `boot::capsule("INSTALLER", "installer", ...)` from
`src/userspace/capsule_installer/`. It prints `[INSTALLER] capsule spawned` through `capsule_boot::boot` on
success, or a `[ERROR]` line with the `SpawnError` (framebuffer under `NONOS_FBCONSOLE=1`). A present
marker means `mk_service_lookup("installer")` resolves, so the market and the desktop can request loads; an
absent one means nothing can install. The failure mode worth understanding is that the installer's own
replies are thin and the real verdict is the kernel's: a `LOAD_BY_NAME` returns the new capsule pid on
success or the negative `rc` from `mk_capsule_load` on failure, so a rejected install shows up as a
negative return whose cause (a certificate that did not chain to the trust anchor, a missing ML-DSA-65
signature, a requested cap outside the manifest) was decided inside the kernel's verified-spawn pipeline,
not here. A load that fails before the syscall is the name validator refusing a name that is empty, over 64
bytes, or outside `[A-Za-z0-9_-]`, and the paid path returns `EAGAIN` when the [payment](payment.md)
service is not reachable.

## Honest gaps

The installer has no caller attestation, its inbox is a public port, so any capsule that can reach it can
request a load, but the kernel's trust chain means a load only succeeds for a properly signed image, so
the exposure is a denial-of-service surface rather than an authenticity one. The installer holds no state,
so there is nothing to leak. (The reject-all `offline-verify` development stub is on the sibling
[market](market.md), which does verify content itself; the installer defers verification entirely and has
no such stub.)

## Source map

```
  userland/capsule_installer/src/server/runner.rs             the loop
  userland/capsule_installer/src/server/handlers/load_by_name.rs   read /capsules artifacts + load
  userland/capsule_installer/src/server/handlers/load_store.rs     load from IPC-supplied blobs
  userland/capsule_installer/src/server/handlers/install.rs        the paid-install path
  userland/capsule_installer/src/protocol/types.rs                 the ops and CapsuleLoadRequest use
```

# Service Capsules

This page documents the non-driver, non-network service capsules in userland:
core storage services, security services, desktop services, policy, market,
proof, power, login, clipboard, image codec, and toolkit. Read
[Capsule Inventory](capsules.md), then [Desktop](desktop.md) and
[Storage](../subsystems/storage.md).

---

## 1. Service loop shape

Most service capsules are no_std binaries with `_start`, heap setup, and a
blocking or polling IPC server loop. The RAMFS, keyring, entropy, crypto,
payment, installer, and VFS runners use `mk_ipc_recv` and reply with
`mk_ipc_send` after decoding and dispatching a request
(`userland/capsule_ramfs/src/server/runner.rs:28`,
`userland/capsule_keyring/src/server/runner.rs:28`,
`userland/capsule_entropy/src/server/runner.rs:27`,
`userland/capsule_crypto/src/server/runner.rs:28`,
`userland/capsule_payment/src/server/runner.rs:27`,
`userland/capsule_installer/src/server/runner.rs:26`,
`userland/capsule_vfs/src/server/runner.rs:27`). Clipboard, attest, and power
use explicit buffers and polling/yield behavior around their own protocol
handlers (`userland/capsule_clipboard/src/server/runner.rs:27`,
`userland/capsule_attest/src/server/runner.rs:27`,
`userland/capsule_power/src/server/runner.rs:28`).

```
  +------------------+
  | service capsule  |
  | _start           |
  +--------+---------+
           |
  +--------+---------+
  | heap setup       |
  | optional setup   |
  +--------+---------+
           |
  +--------+---------+
  | ipc recv         |
  | decode request   |
  +--------+---------+
           |
  +--------+---------+
  | dispatch handler |
  | ipc reply        |
  +------------------+
```

## 2. Core, storage, and security services

| Capsule | Service contract | Protocol ops | Runner |
|---------|------------------|--------------|--------|
| `ramfs` | RAM filesystem service with open, close, read, write, and truncate operations. | `userland/capsule_ramfs/src/protocol/types.rs:17` to `userland/capsule_ramfs/src/protocol/types.rs:21` | `userland/capsule_ramfs/src/server/runner.rs:28`, dispatch at `userland/capsule_ramfs/src/server/dispatch.rs:26` |
| `vfs` | VFS service with open, close, read, write, stat, list, healthcheck, mkdir, unlink, and rename operations. | `userland/capsule_vfs/src/protocol/types.rs:20` to `userland/capsule_vfs/src/protocol/types.rs:29` | `userland/capsule_vfs/src/server/runner.rs:27`, dispatch at `userland/capsule_vfs/src/server/dispatch.rs:26` |
| `keyring` | Key store and wallet signing service with store, retrieve, delete, lock, unlock, metadata, count, wallet import, wallet generate, wallet address, NOX receipt signing, and NOX approve signing operations. | `userland/capsule_keyring/src/protocol/types.rs:17` to `userland/capsule_keyring/src/protocol/types.rs:28` | `userland/capsule_keyring/src/server/runner.rs:28`, dispatch at `userland/capsule_keyring/src/server/dispatch.rs:27` |
| `entropy` | Entropy service with get random, get stats, reseed, and healthcheck operations. | `userland/capsule_entropy/src/protocol/types.rs:30` to `userland/capsule_entropy/src/protocol/types.rs:33` | `userland/capsule_entropy/src/server/runner.rs:27`, dispatch at `userland/capsule_entropy/src/server/dispatch.rs:25` |
| `crypto` | Crypto service with BLAKE3, SHA3-256, healthcheck, SHA-256, SHA-512, Ed25519 verify, ChaCha20-Poly1305 seal and open, AES-256-GCM seal and open, plus X25519, HMAC-SHA256, and HKDF-SHA256 operations. | `userland/capsule_crypto/src/protocol/types.rs:20` to `userland/capsule_crypto/src/protocol/types.rs:29`, `userland/capsule_crypto/src/protocol/primitives.rs:17` to `userland/capsule_crypto/src/protocol/primitives.rs:20` | `userland/capsule_crypto/src/server/runner.rs:28`, dispatch at `userland/capsule_crypto/src/server/dispatch.rs:27` |
| `attest` | Attestation service with healthcheck, proof summary, proof invariants, proof boot, and proof capsule list operations. | `userland/capsule_attest/src/protocol/ops.rs:17` to `userland/capsule_attest/src/protocol/ops.rs:21` | `userland/capsule_attest/src/server/runner.rs:27`, router at `userland/capsule_attest/src/server/handlers/router.rs:29` |
| `policy` | Policy service with get and set operations over typed fields. | `userland/policy_proto/src/ops.rs:17`, fields at `userland/policy_proto/src/field.rs:17` | `userland/capsule_policy/src/server/runner.rs:23` |
| `market` | Market index service with load index, list apps, get app, get release, install ready, and healthcheck operations. | `userland/capsule_market/src/protocol/ops.rs:17` to `userland/capsule_market/src/protocol/ops.rs:22` | `userland/capsule_market/src/server/runner.rs:32` |
| `installer` | Installer service with healthcheck and install operations. | `userland/capsule_installer/src/protocol/types.rs:17` to `userland/capsule_installer/src/protocol/types.rs:18` | `userland/capsule_installer/src/server/runner.rs:26`, dispatch at `userland/capsule_installer/src/server/dispatch.rs:22` |
| `payment` | Payment service with healthcheck, pay, and drain receipts operations. | `userland/capsule_payment/src/protocol/types.rs:17` to `userland/capsule_payment/src/protocol/types.rs:19` | `userland/capsule_payment/src/server/runner.rs:27`, dispatch at `userland/capsule_payment/src/server/dispatch.rs:25` |
| `power` | Power service with healthcheck, reboot, and shutdown operations. | `userland/capsule_power/src/protocol/ops.rs:17` to `userland/capsule_power/src/protocol/ops.rs:19` | `userland/capsule_power/src/server/runner.rs:28`, router at `userland/capsule_power/src/server/handlers/router.rs:27` |
| `proof_io` | Direct syscall proof capsule that checks time calls, invalid syscall number handling, invalid pointer handling, invalid size handling, retired syscall rejection, then emits a pass or fail debug line. | direct syscall calls | `userland/capsule_proof_io/src/main.rs:37`, retired syscall list at `userland/capsule_proof_io/src/main.rs:24` |

## 3. Desktop service capsules

| Capsule | Service contract | Protocol ops | Runner |
|---------|------------------|--------------|--------|
| `compositor` | Scene, damage, focus, input subscription, cursor, scene removal, and display-info service. | `userland/compositor/src/protocol/ops.rs:17` to `userland/compositor/src/protocol/ops.rs:24` | `userland/compositor/src/server/runner/entry.rs:23`, dispatch at `userland/compositor/src/server/runner/dispatch.rs:24` |
| `wm` | Window lifecycle, geometry, focus, z-order, topmost query, routed focus, and lifecycle subscription service. | `userland/capsule_wm/src/protocol/ops.rs:17` to `userland/capsule_wm/src/protocol/ops.rs:29` | `userland/capsule_wm/src/server/runner/run.rs:28` |
| `input_router` | Input subscription and grab service that routes kernel input events to shell or focused windows. | `userland/capsule_input_router/src/protocol/ops.rs:17` to `userland/capsule_input_router/src/protocol/ops.rs:20` | `userland/capsule_input_router/src/server/runner.rs:30` |
| `desktop_shell` | Shell service with healthcheck, tray register, tray update, tray remove, notify, and spotlight open operations. | `userland/capsule_desktop_shell/src/protocol/ops.rs:17` to `userland/capsule_desktop_shell/src/protocol/ops.rs:22` | `userland/capsule_desktop_shell/src/server/runner/run.rs:27` |
| `wallpaper` | Wallpaper service with healthcheck, set wallpaper, get wallpaper, set policy, and fade operations. | `userland/capsule_wallpaper/src/protocol/ops.rs:17` to `userland/capsule_wallpaper/src/protocol/ops.rs:21` | `userland/capsule_wallpaper/src/main.rs:37` |
| `wallpaper_catalog` | Wallpaper catalog service with count, size, chunk, and slug operations. | `userland/capsule_wallpaper_catalog/src/protocol/ops.rs:17` to `userland/capsule_wallpaper_catalog/src/protocol/ops.rs:20` | `userland/capsule_wallpaper_catalog/src/main.rs:30` |
| `image_codec` | Image decode service with healthcheck, PNG, BMP, raw LZ4, and JPEG decode operations. | `userland/capsule_image_codec/src/protocol/ops.rs:17` to `userland/capsule_image_codec/src/protocol/ops.rs:21` | `userland/capsule_image_codec/src/server/runner.rs:28` |
| `clipboard` | Clipboard service with healthcheck, copy, paste, history list, history get, clear, and idle timeout operations. | `userland/capsule_clipboard/src/protocol/ops.rs:17` to `userland/capsule_clipboard/src/protocol/ops.rs:23` | `userland/capsule_clipboard/src/server/runner.rs:27`, router at `userland/capsule_clipboard/src/server/handlers/router.rs:30` |
| `login` | Login session service with healthcheck, start session, end session, and get state operations. | `userland/capsule_login/src/protocol/ops.rs:1` to `userland/capsule_login/src/protocol/ops.rs:4` | `userland/capsule_login/src/server/runner.rs:16` |
| `toolkit` | Toolkit service with healthcheck, theme apply, animation tick, component render, and theme get operations. | `userland/toolkit/src/protocol/ops.rs:19` to `userland/toolkit/src/protocol/ops.rs:23` | `userland/toolkit/src/server/runner.rs:29`, dispatch at `userland/toolkit/src/server/dispatch.rs:25` |

## 4. Policy fields

Policy requests use a 12-byte header with op, field, kind, status, and payload
length (`userland/policy_proto/src/hdr.rs:17`). The supported fields include
desktop fields such as brightness, mouse sensitivity, sound enabled, anonymous
mode, Nym enabled, theme, keyboard layout, timeout, language, font size,
cursor size, and wallpaper, kernel fields such as ASLR, stack guard, NX, SMEP,
SMAP, debug, serial, watchdog, preempt, hugepages, IOMMU, and seccomp, plus
hostname and domain name (`userland/policy_proto/src/field.rs:20`). The policy
runner decodes the header, validates payload length, decodes the field, and
dispatches `OP_GET` or `OP_SET` (`userland/capsule_policy/src/server/runner.rs:36`).

## 5. Boot inclusion

Core after RAMFS starts keyring, entropy, crypto, and policy
(`src/userspace/init/spawn_plan/core.rs:22`). VFS starts as its own phase
(`src/userspace/init/spawn_plan/core.rs:29`). Desktop services start image
codec, clipboard, attest, login, and toolkit
(`src/userspace/init/spawn_plan/desktop_services.rs:17`). The market service is
feature gated and is spawned from the core plan
(`src/userspace/init/spawn_plan/core.rs:35`).


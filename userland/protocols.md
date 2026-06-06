# Userland Protocol Atlas

This page describes the IPC and control protocols used by NØNOS userland
capsules. Read [Userland Model](README.md), [Capsule Inventory](capsules.md),
and [GUI Contracts](gui-contracts.md) first.

The inventory page answers which capsule owns which endpoint. This page answers
what each protocol does after a message reaches that endpoint.

---

## 1. Dispatch shape

Service capsules follow a small shape: receive a frame, parse the request,
dispatch by `op`, and return protocol errors for unknown or malformed input. The
L2 network runner shows the blocking service loop: it receives from inbox `0`,
parses the frame, matches the op, calls a handler, and emits `E_BAD_OP` for
unknown operations (`userland/capsule_net_l2/src/server/runner.rs:31`,
`userland/capsule_net_l2/src/server/runner.rs:44`,
`userland/capsule_net_l2/src/server/runner.rs:51`). The compositor and WM use
the same explicit match style, with empty-body bad ops mapped to `E_BAD_OP` and
non-empty malformed ops mapped to `E_INVAL`
(`userland/compositor/src/server/runner/dispatch.rs:31`,
`userland/compositor/src/server/runner/dispatch.rs:44`,
`userland/capsule_wm/src/server/runner/dispatch.rs:32`,
`userland/capsule_wm/src/server/runner/dispatch.rs:50`). The input router drains
nonblocking IPC from its service inbox and applies the same error split
(`userland/capsule_input_router/src/server/drain_ipc.rs:28`,
`userland/capsule_input_router/src/server/drain_ipc.rs:51`).

```
+------------------+
| caller capsule   |
+--------+---------+
         |
+--------+---------+
| service inbox    |
| recv frame       |
+--------+---------+
         |
+--------+---------+
| parse header     |
| split body       |
+--------+---------+
         |
+--------+---------+
| match op         |
| call handler     |
+--------+---------+
         |
+--------+---------+
| reply status     |
| reply payload    |
+------------------+
```

## 2. Desktop protocols

The GUI stack is split across compositor, WM, input router, desktop shell,
wallpaper services, image codec, clipboard, login, and toolkit. Each capsule has
its own op table instead of sharing a desktop super-protocol.

| Capsule | Protocol surface | Dispatch evidence |
|---------|------------------|-------------------|
| `compositor` | healthcheck, scene submit, damage commit, focus set, input subscribe, cursor update, scene remove, display info | Ops are declared at `userland/compositor/src/protocol/ops.rs:17` to `userland/compositor/src/protocol/ops.rs:24`, dispatch matches them at `userland/compositor/src/server/runner/dispatch.rs:31`. |
| `wm` | healthcheck, window open, close, move, resize, focus, raise, lifecycle subscribe, minimize, restore, query topmost, route focus, query focus | Ops are declared at `userland/capsule_wm/src/protocol/ops.rs:17` to `userland/capsule_wm/src/protocol/ops.rs:29`, dispatch matches them at `userland/capsule_wm/src/server/runner/dispatch.rs:32`. |
| `input_router` | healthcheck, subscribe, grab request, grab release | Ops are declared at `userland/capsule_input_router/src/protocol/ops.rs:17` to `userland/capsule_input_router/src/protocol/ops.rs:20`, IPC drain routes them at `userland/capsule_input_router/src/server/drain_ipc.rs:44`. |
| `desktop_shell` | healthcheck, tray register, tray update, tray remove, notify, spotlight open | Ops are declared at `userland/capsule_desktop_shell/src/protocol/ops.rs:17` to `userland/capsule_desktop_shell/src/protocol/ops.rs:22`. |
| `wallpaper` | healthcheck, set wallpaper, get wallpaper, set policy, fade | Ops are declared at `userland/capsule_wallpaper/src/protocol/ops.rs:17` to `userland/capsule_wallpaper/src/protocol/ops.rs:21`. |
| `wallpaper_catalog` | get count, get size, get chunk, get slug | Ops are declared at `userland/capsule_wallpaper_catalog/src/protocol/ops.rs:17` to `userland/capsule_wallpaper_catalog/src/protocol/ops.rs:20`. |
| `image_codec` | healthcheck, decode PNG, decode BMP, decode LZ4 raw, decode JPEG | Ops are declared at `userland/capsule_image_codec/src/protocol/ops.rs:17` to `userland/capsule_image_codec/src/protocol/ops.rs:21`. |
| `clipboard` | healthcheck, copy, paste, history list, history get, clear, set idle timeout | Ops are declared at `userland/capsule_clipboard/src/protocol/ops.rs:17` to `userland/capsule_clipboard/src/protocol/ops.rs:23`. |
| `login` | healthcheck, start session, end session, get state | Ops are declared at `userland/capsule_login/src/protocol/ops.rs:1` to `userland/capsule_login/src/protocol/ops.rs:4`. |
| `toolkit` | healthcheck, theme apply, animation tick, component render, theme get | Ops and payload limits are declared at `userland/toolkit/src/protocol/ops.rs:17` to `userland/toolkit/src/protocol/ops.rs:26`. |

## 3. App runner protocol

Apps built on `nonos_app_skeleton` do not implement a custom service protocol by
default. The shared runner initializes heap, discovers peers, constructs the app,
boots the window, allocates an IPC receive buffer, and enters a frame loop
(`userland/app_skeleton/src/runner/entry.rs:30`,
`userland/app_skeleton/src/runner/entry.rs:46`). Boot opens the WM window,
subscribes to input, and primes the first frame
(`userland/app_skeleton/src/runner/boot.rs:33`,
`userland/app_skeleton/src/runner/boot.rs:39`,
`userland/app_skeleton/src/runner/boot.rs:40`,
`userland/app_skeleton/src/runner/boot.rs:41`).

Each service frame refreshes input, ensures the first frame is primed, drains
pending IPC, closes when requested, repaints when requested, then waits for
display vsync (`userland/app_skeleton/src/runner/service_frame.rs:34`,
`userland/app_skeleton/src/runner/service_frame.rs:39`,
`userland/app_skeleton/src/runner/service_frame.rs:47`,
`userland/app_skeleton/src/runner/service_frame.rs:50`,
`userland/app_skeleton/src/runner/service_frame.rs:53`).

The app receive loop accepts two frame classes. `NCTL` is a desktop-shell control
frame used for focus, and `NINP` is input delivery from the router. The control
path only honors a focus-self request if the sender pid resolves to
`desktop_shell` (`userland/app_skeleton/src/runner/control.rs:22`,
`userland/app_skeleton/src/runner/control.rs:35`,
`userland/app_skeleton/src/runner/control.rs:39`,
`userland/app_skeleton/src/runner/control.rs:59`). Input delivery requires the
`NINP` magic and then decodes the event body
(`userland/app_skeleton/src/runner/dispatch.rs:20`,
`userland/app_skeleton/src/runner/dispatch.rs:27`,
`userland/app_skeleton/src/runner/dispatch.rs:31`).

The app drain loop reads from inbox `0`, handles control frames first, parses
input, normalizes decoration events, raises focus on click, treats decoration
close as close, and passes the event to the app's `on_event`
(`userland/app_skeleton/src/runner/drain_ipc.rs:31`,
`userland/app_skeleton/src/runner/drain_ipc.rs:47`,
`userland/app_skeleton/src/runner/drain_ipc.rs:50`,
`userland/app_skeleton/src/runner/drain_ipc.rs:51`,
`userland/app_skeleton/src/runner/drain_ipc.rs:52`,
`userland/app_skeleton/src/runner/drain_ipc.rs:53`,
`userland/app_skeleton/src/runner/drain_ipc.rs:56`). Close removes the scene,
unsubscribes from input, releases the surface, unmaps the backing memory, closes
the WM window, and exits (`userland/app_skeleton/src/runner/teardown.rs:31`,
`userland/app_skeleton/src/runner/teardown.rs:35`,
`userland/app_skeleton/src/runner/teardown.rs:36`,
`userland/app_skeleton/src/runner/teardown.rs:39`,
`userland/app_skeleton/src/runner/teardown.rs:42`,
`userland/app_skeleton/src/runner/teardown.rs:46`).

```
+----------------+
| app start      |
+-------+--------+
        |
+-------+--------+
| open WM window |
| subscribe input|
+-------+--------+
        |
+-------+--------+
| prime frame    |
+-------+--------+
        |
+-------+--------+
| drain NCTL     |
| drain NINP     |
+-------+--------+
        |
+-------+--------+
| repaint or exit|
+----------------+
```

## 4. Launch and focus frames

Desktop launcher entries are data, not hardcoded drawing side effects. Each
launcher entry carries icon, launch id, label, and service name
(`userland/capsule_desktop_shell/src/state/apps.rs:28`,
`userland/capsule_desktop_shell/src/state/apps.rs:35`). A launcher request looks
up the target service. If a pid exists, it sends focus control directly to that
pid. If no pid exists, it sends a launch request to `desktop.launcher`
(`userland/capsule_desktop_shell/src/server/handlers/launcher_request/request.rs:19`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_request/lookup_pid.rs:19`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_request/focus.rs:19`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_request/launch.rs:19`).

| Frame | Size | Producer | Consumer | Fields |
|-------|------|----------|----------|--------|
| `NCTL` | 8 bytes | desktop shell | app skeleton | magic `NCTL`, version `1`, op `1` for focus self. Constants are at `userland/capsule_desktop_shell/src/server/handlers/launcher_request/constants.rs:17` to `userland/capsule_desktop_shell/src/server/handlers/launcher_request/constants.rs:20`, frame encoding is at `userland/capsule_desktop_shell/src/server/handlers/launcher_request/focus_frame.rs:17`. |
| `NLAU` | 8 bytes | desktop shell | init launcher broker | magic `NLAU`, version `1`, launch id. Constants are at `userland/capsule_desktop_shell/src/server/handlers/launcher_request/constants.rs:21` to `userland/capsule_desktop_shell/src/server/handlers/launcher_request/constants.rs:23`, frame encoding is at `userland/capsule_desktop_shell/src/server/handlers/launcher_request/launch_frame.rs:17`. |

The init broker has matching `desktop.launcher`, port `4700`, `NLAU`, and
version constants (`src/userspace/init/launcher/constants.rs:17`,
`src/userspace/init/launcher/constants.rs:18`,
`src/userspace/init/launcher/constants.rs:19`,
`src/userspace/init/launcher/constants.rs:20`). Its decoder rejects any frame
whose length, magic, or version does not match, then returns the launch id
(`src/userspace/init/launcher/decode.rs:17`,
`src/userspace/init/launcher/decode.rs:21`,
`src/userspace/init/launcher/decode.rs:23`,
`src/userspace/init/launcher/decode.rs:26`).

## 5. Core service protocols

The core services use small, fixed op tables. Some use an 8-byte local header,
while entropy, crypto, and VFS use a 20-byte v1 frame with magic, version, op,
flags, request id, and payload length.

| Capsule | Protocol surface | Source |
|---------|------------------|--------|
| `ramfs` | open, close, read, write, truncate, plus create and truncate open flags | `userland/capsule_ramfs/src/protocol/types.rs:17` to `userland/capsule_ramfs/src/protocol/types.rs:28` |
| `vfs` | magic `NOVF`, version `1`, open, close, read, write, stat, list, healthcheck, mkdir, unlink, rename, create/truncate/append flags, path and payload caps | `userland/capsule_vfs/src/protocol/types.rs:17` to `userland/capsule_vfs/src/protocol/types.rs:44` |
| `keyring` | store, retrieve, delete, lock, unlock, metadata, count, wallet import, wallet generate, wallet address, sign NOX receipt, sign NOX approve | `userland/capsule_keyring/src/protocol/types.rs:17` to `userland/capsule_keyring/src/protocol/types.rs:32` |
| `entropy` | magic `NOEN`, version `1`, get random, get stats, reseed, healthcheck, bounded random and reseed sizes | `userland/capsule_entropy/src/protocol/types.rs:26` to `userland/capsule_entropy/src/protocol/types.rs:54` |
| `crypto` | magic `NOCX`, version `1`, BLAKE3, SHA3-256, SHA-256, SHA-512, Ed25519 verify, ChaCha20-Poly1305 seal/open, AES-256-GCM seal/open, bounded verify and AEAD payload sizes | `userland/capsule_crypto/src/protocol/types.rs:17` to `userland/capsule_crypto/src/protocol/types.rs:66` |
| `crypto` primitives | X25519 public key, X25519 shared secret, HMAC-SHA256, HKDF-SHA256, primitive-specific caps | `userland/capsule_crypto/src/protocol/primitives.rs:17` to `userland/capsule_crypto/src/protocol/primitives.rs:25` |
| `attest` | healthcheck, proof summary, proof invariants, proof boot, proof capsule list | `userland/capsule_attest/src/protocol/ops.rs:17` to `userland/capsule_attest/src/protocol/ops.rs:21` |
| `policy` | get and set over typed policy fields | `userland/policy_proto/src/ops.rs:17` to `userland/policy_proto/src/ops.rs:18`, fields at `userland/policy_proto/src/field.rs:17` to `userland/policy_proto/src/field.rs:57` |
| `market` | load index, list apps, get app, get release, install ready, healthcheck | `userland/capsule_market/src/protocol/ops.rs:17` to `userland/capsule_market/src/protocol/ops.rs:22` |
| `installer` | healthcheck, install, kernel reply endpoint, 8-byte header, `EINVAL`, `EAGAIN` | `userland/capsule_installer/src/protocol/types.rs:17` to `userland/capsule_installer/src/protocol/types.rs:25` |
| `payment` | healthcheck, pay, drain receipts, kernel reply endpoint, 8-byte header, `EINVAL`, `EAGAIN` | `userland/capsule_payment/src/protocol/types.rs:17` to `userland/capsule_payment/src/protocol/types.rs:26` |
| `power` | healthcheck, reboot, shutdown | `userland/capsule_power/src/protocol/ops.rs:17` to `userland/capsule_power/src/protocol/ops.rs:19` |

## 6. Network protocols

The network stack is capsule-separated by layer. L2 owns link and frame
operations, IP owns packet and route operations, UDP/TCP own transport sessions,
DNS owns name resolution, sockets owns a socket-style facade, and Nym owns
privacy session operations.

| Capsule | Protocol surface | Source |
|---------|------------------|--------|
| `net.l2` | healthcheck, get MAC, get link, send frame, poll frame, ARP resolve | `userland/capsule_net_l2/src/protocol/ops.rs:21` to `userland/capsule_net_l2/src/protocol/ops.rs:26` |
| `net.ip` | healthcheck, get config, set config, send packet, poll packet, route add, route clear | `userland/capsule_net_ip/src/protocol/ops.rs:21` to `userland/capsule_net_ip/src/protocol/ops.rs:27` |
| `net.udp` | healthcheck, bind, unbind, send, receive | `userland/capsule_net_udp/src/protocol/ops.rs:17` to `userland/capsule_net_udp/src/protocol/ops.rs:21` |
| `net.dhcp.client` | healthcheck, lease request, lease status, lease release, lease renew | `userland/capsule_net_dhcp/src/protocol/ops.rs:17` to `userland/capsule_net_dhcp/src/protocol/ops.rs:21` |
| `net.tcp` | healthcheck, listen, connect, accept, send, receive, close, shutdown | `userland/capsule_net_tcp/src/protocol/ops.rs:17` to `userland/capsule_net_tcp/src/protocol/ops.rs:24` |
| `net.dns` | healthcheck, resolve A, resolve AAAA, flush cache, set upstream | `userland/capsule_net_dns/src/protocol/ops.rs:17` to `userland/capsule_net_dns/src/protocol/ops.rs:21` |
| `net.sockets` | healthcheck, socket, bind, listen, accept, connect, send, receive, close, getsockopt, setsockopt | `userland/capsule_net_sockets/src/protocol/ops.rs:17` to `userland/capsule_net_sockets/src/protocol/ops.rs:27` |
| `net.nym` | healthcheck, gateway, session open, send, receive, cover traffic tick, close, topology, credential, SURB, reply, timing, authority, directory sync, status | `userland/capsule_net_nym/src/protocol/ops.rs:17` to `userland/capsule_net_nym/src/protocol/ops.rs:32` |

`net.sockets` uses the same receive, parse, dispatch, and respond pattern as
other services, but delegates the op match into its handler module
(`userland/capsule_net_sockets/src/server/runner.rs:28`,
`userland/capsule_net_sockets/src/server/runner.rs:37`,
`userland/capsule_net_sockets/src/server/runner.rs:38`,
`userland/capsule_net_sockets/src/server/runner.rs:39`).

## 7. Driver protocols

Drivers are user-mode capsules with their own op tables. They are not kernel
inline drivers. Hardware access is granted through capability bits and brokered
mapping paths, while the public control surface is still the capsule IPC
protocol documented here and in [Drivers](drivers.md).

| Driver family | Capsules | Protocol surface |
|---------------|----------|------------------|
| Virtio random | `driver.virtio_rng` | fill random and healthcheck at `userland/capsule_driver_virtio_rng/src/protocol/ops.rs:21` to `userland/capsule_driver_virtio_rng/src/protocol/ops.rs:22`. |
| Virtio block | `driver.virtio_blk0` | healthcheck, capacity, read blocks, write blocks, flush at `userland/capsule_driver_virtio_blk/src/protocol/ops.rs:16` to `userland/capsule_driver_virtio_blk/src/protocol/ops.rs:20`. |
| Virtio network | `driver.virtio_net0` | healthcheck, link status, MAC address, TX packet, RX packet at `userland/capsule_driver_virtio_net/src/protocol/ops.rs:21` to `userland/capsule_driver_virtio_net/src/protocol/ops.rs:25`. |
| Virtio GPU | `driver.virtio_gpu0` | healthcheck, controller info, display info, control queue state, caps query, resource creation, backing attach, transfer to host, scanout, flush, mode list, primary surface at `userland/capsule_driver_virtio_gpu/src/protocol/ops.rs:16` to `userland/capsule_driver_virtio_gpu/src/protocol/ops.rs:27`. |
| USB host | `driver.xhci0` | healthcheck, controller status, port status, slot control, address device, descriptor reads, transfer ring allocation, control transfer, interrupt in at `userland/capsule_driver_xhci/src/protocol/ops.rs:16` to `userland/capsule_driver_xhci/src/protocol/ops.rs:26`. |
| Input devices | `driver.ps2_kbd0`, `driver.usb_hid0`, `driver.i2c_hid0` | PS/2 exposes event and mouse polling at `userland/capsule_driver_ps2_input/src/protocol/ops.rs:16` to `userland/capsule_driver_ps2_input/src/protocol/ops.rs:20`, USB HID exposes keyboard and mouse report feed/poll/state at `userland/capsule_driver_usb_hid/src/protocol/ops.rs:17` to `userland/capsule_driver_usb_hid/src/protocol/ops.rs:23`, I2C HID exposes healthcheck, probe, descriptor at `userland/capsule_driver_i2c_hid/src/protocol/ops.rs:1` to `userland/capsule_driver_i2c_hid/src/protocol/ops.rs:3`. |
| NICs | `driver.e1000_0`, `driver.rtl8139_0`, `driver.rtl8169_0` | E1000 exposes link, MAC, TX, RX, stats at `userland/capsule_driver_e1000/src/protocol/ops.rs:23` to `userland/capsule_driver_e1000/src/protocol/ops.rs:28`, RTL8139 and RTL8169 expose the same surface at `userland/capsule_driver_rtl8139/src/protocol/ops.rs:17` to `userland/capsule_driver_rtl8139/src/protocol/ops.rs:22` and `userland/capsule_driver_rtl8169/src/protocol/ops.rs:17` to `userland/capsule_driver_rtl8169/src/protocol/ops.rs:22`. |
| Storage controllers | `driver.ahci0`, `driver.nvme0`, `driver.usb_msc0` | AHCI exposes healthcheck, controller info, port list at `userland/capsule_driver_ahci/src/protocol/ops.rs:17` to `userland/capsule_driver_ahci/src/protocol/ops.rs:19`, NVMe exposes identify and SMART health at `userland/capsule_driver_nvme/src/protocol/ops.rs:17` to `userland/capsule_driver_nvme/src/protocol/ops.rs:21`, USB MSC exposes probe, command builders, CSW accept, and state at `userland/capsule_driver_usb_msc/src/protocol/ops.rs:17` to `userland/capsule_driver_usb_msc/src/protocol/ops.rs:24`. |
| Audio and buses | `driver.hda0`, `driver.i2c_pci0`, `driver.iwlwifi0` | HDA exposes controller, codec, and stream layout queries at `userland/capsule_driver_hda/src/protocol/ops.rs:17` to `userland/capsule_driver_hda/src/protocol/ops.rs:21`, I2C PCI exposes controller info, register snapshot, timing info, transfer, probe at `userland/capsule_driver_i2c_pci/src/protocol/ops.rs:1` to `userland/capsule_driver_i2c_pci/src/protocol/ops.rs:6`, IWLWiFi exposes device, firmware, RF, DMA, firmware stage, and alive wait at `userland/capsule_driver_iwlwifi/src/protocol/ops.rs:9` to `userland/capsule_driver_iwlwifi/src/protocol/ops.rs:15`. |

## 8. Proof and diagnostic capsules

Not every userland binary is a long-lived service protocol. `proof_io` is a
direct syscall proof program: it loops over `mk_time_millis`, checks unknown
syscall tags, invalid debug pointers, invalid debug sizes, retired syscall tags,
prints a pass or fail message, and exits
(`userland/capsule_proof_io/src/main.rs:36`,
`userland/capsule_proof_io/src/main.rs:38`,
`userland/capsule_proof_io/src/main.rs:44`,
`userland/capsule_proof_io/src/main.rs:48`,
`userland/capsule_proof_io/src/main.rs:52`,
`userland/capsule_proof_io/src/main.rs:56`,
`userland/capsule_proof_io/src/main.rs:64`).

`input_proof` is a GUI app using the app skeleton, not a separate IPC service
protocol. Its `_start` calls `nonos_app_skeleton::run`, and its app
implementation records input markers and paints a proof surface
(`userland/capsule_input_proof/src/main.rs:24`,
`userland/capsule_input_proof/src/main.rs:27`,
`userland/capsule_input_proof/src/proof/app.rs:36`,
`userland/capsule_input_proof/src/proof/app.rs:41`,
`userland/capsule_input_proof/src/proof/app.rs:46`).

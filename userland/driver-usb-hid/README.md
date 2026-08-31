# The USB HID Driver Capsule

`capsule_driver_usb_hid` is the USB HID class driver in the NONOS tree: a signed userland capsule
that turns USB keyboard and mouse HID reports into `InputEvent`s and posts them into the kernel input
ring. It sits on top of the xHCI transport capsule (`driver.xhci0`), which owns the controller
mechanics; this capsule owns HID class parsing, boot-report normalization, and the input-post path.
Its source is organized into a small set of pillars, and this documentation mirrors that structure so
a page can be read beside the folder it describes.

## Identity

| Field | Value | Source |
|---|---|---|
| Slug | `driver-usb-hid` | `userland/capsule_driver_usb_hid/Capsule.mk:6` |
| Service handle | `driver.usb_hid0` | `Capsule.mk:7`, `src/userspace/capsule_driver_usb_hid/spawn.rs:31` |
| Namespace | `systems.nonos.driver.usb_hid0` | `Capsule.mk:12` |
| Service endpoint | `service:4222:driver.usb_hid0` | `Capsule.mk:13`, `spawn.rs:32` |
| Reply endpoint | `reply:4223:endpoint.4294967314` | `Capsule.mk:14`, `spawn.rs:33`, `spawn.rs:34` |
| Binary name | `driver_usb_hid` | `Capsule.mk:10` |
| Capability mask | `0x200019` | `Capsule.mk:15`, `spawn.rs:51` |
| Kernel mirror | `src/userspace/capsule_driver_usb_hid` | `Capsule.mk:9` |

The mask `0x200019` decomposes bit by bit against `src/capabilities/types/bit.rs` (the enum is in `src/capabilities/types/defs.rs`):

```
  0x000001  CoreExec      run as a process              1        types/bit.rs:23
  0x000008  IPC           send/recv on its endpoints    8        types/bit.rs:26
  0x000010  Memory        map its own heap and stack    16       types/bit.rs:27
  0x200000  InputSource   post into the input ring      2097152  types/bit.rs:44
  --------
  0x200019  = 1 + 8 + 16 + 2097152
```

The kernel spawn path requests exactly those four capabilities and no others, by name:
`Capability::CoreExec | IPC | Memory | InputSource`
(`src/userspace/capsule_driver_usb_hid/spawn.rs:51`). The one that matters is `InputSource`: it is
what lets this capsule post into the shared input ring. The gate is explicit.
`MkInputEventPost` is admitted only when the token satisfies `can_input_source`
(`src/syscall/contract/cap_table/mk.rs:78`), and `can_input_source` grants only to a holder of
`InputSource`, `Irq`, or `Admin` (`src/capabilities/token/types.rs:166`). This capsule holds
`InputSource` alone, so its authority to post input rests on that one bit.

There is no `Network` (4), no `FileSystem` (64), and, crucially, no `Driver` (65536), `DeviceEnum`
(32768), `Mmio` (131072), `Irq` (262144), `Dma` (524288), or `Pio` (1048576) bit in the mask
(`src/capabilities/types/bit.rs:39`). The capsule cannot enumerate PCI devices, claim a controller
through the [hardware broker](../../subsystems/hardware-broker/claim.md), map a register through
[MMIO](../../subsystems/hardware-broker/mmio.md), bind an interrupt through
[IRQ](../../subsystems/hardware-broker/irq.md), or allocate DMA. Every hardware effect it needs is an
IPC call to `driver.xhci0`, which holds those grants.

One correction. The capsule's own top-of-file `Capsule.mk` comment is accurate, but earlier prose in
the tree described the mask as `0x18` (`IPC | Memory`) and stated the capsule makes no input-post
call. That predates the input-post path. The shipping `Capsule.mk` and the kernel spawn mirror both
request `0x200019` including `InputSource`, and the code does post: `mk_input_event_post` is called
from `src/hid/post_wire.rs:22`. The `0x200019` mask, `InputSource`, and the post path are the shipping
truth; the `0x18`/no-post description is stale.

## The pillars

The source under `userland/capsule_driver_usb_hid/src/` has two faces that share one `State`. A live
driver face resolves the xHCI transport, enumerates HID devices over IPC, drains their interrupt IN
endpoints, and posts each parsed report into the kernel input ring. A request/reply service face
answers `driver.usb_hid0` for diagnostics and offline feeds. The report parsers and the input-post
path are shared by both.

```
  driver.xhci0  ->  orchestrator/  ->  hid/       ->  kernel input ring
  (transport)      discovery +        report parse    mk_input_event_post
                   poll loop          + input-post

  service caller ->  server/    ->  hid/    (same parsers, same post path)
  (driver.usb_hid0)  dispatch       feed
```

| Page | Mirrors | What it covers |
|---|---|---|
| [protocol.md](protocol.md) | `src/protocol/`, `src/server/` | The `NUHI` service wire format, the seven ops, the dispatch, and the request handlers. |
| [enumeration.md](enumeration.md) | `src/orchestrator/`, `src/descriptors/`, `src/xhci/` | Bring-up: the `driver.xhci0` client, port scan, slot and address, the config-descriptor walk and HID classification, boot-protocol binding, and the cooperative poll loop. |
| [input-post.md](input-post.md) | `src/hid/` | Report normalization and the input-post path: the keyboard diff and keymap, the mouse and tablet parse, and the `post_key`/`post_mouse`/`post_wire` wire into `mk_input_event_post`. |
| [contributing.md](contributing.md) | the whole tree | Where to work, adding an op or extending the parse, and the build and sign steps. |
| [debugging.md](debugging.md) | runtime | The boot and input markers and the failure modes: device not enumerated, no input, wrong keycodes. |

## The operations

The service exposes seven operations, defined once in `src/protocol/ops.rs:17` and dispatched by op
code in `src/server/dispatch.rs:22`:

| Op | Code | Input | Output | Handler |
|---|---|---|---|---|
| `OP_HEALTHCHECK` | `0x0001` | empty | status word | `dispatch.rs:23`, `handlers/health.rs:20` |
| `OP_PROBE_CONFIG` | `0x0002` | raw USB config descriptor (<= 512 B) | binding count then 8-byte records | `dispatch.rs:24`, `handlers/probe_config.rs:24` |
| `OP_FEED_KEYBOARD_REPORT` | `0x0003` | 8-byte boot keyboard report | status word | `dispatch.rs:25`, `handlers/feed_key.rs:21` |
| `OP_FEED_MOUSE_REPORT` | `0x0004` | 3 or 4-byte boot mouse report | status word | `dispatch.rs:26`, `handlers/feed_mouse.rs:21` |
| `OP_POLL_KEYS` | `0x0005` | empty | count then 8-byte key events (<= 16) | `dispatch.rs:27`, `handlers/poll_keys.rs:21` |
| `OP_POLL_MOUSE` | `0x0006` | empty | count then 8-byte mouse events (<= 16) | `dispatch.rs:28`, `handlers/poll_mouse.rs:21` |
| `OP_GET_STATE` | `0x0007` | empty | 48-byte counter block | `dispatch.rs:29`, `handlers/get_state.rs:21` |

These ops are a diagnostic and offline-feed surface. The event stream that reaches the desktop does
not go through `OP_POLL_KEYS` or `OP_POLL_MOUSE`; it goes through the input-post path, which is a
syscall (`mk_input_event_post`), not an IPC reply. The [protocol](protocol.md) page covers the ops in
detail and the [input-post](input-post.md) page covers the post path.

## Lifecycle

The driver is spawned through verified spawn: its signature, `nonos-id` cert, manifest, and
attestation trailer are checked against the baked trust anchor, its four requested capabilities are
held against its manifest ceiling, and only then is its ELF mapped
(`src/userspace/capsule_driver_usb_hid/spawn.rs:37`). `_start` initializes the heap and calls
`orchestrator::run`, which never returns (`src/main.rs:33`). That loop blocks until `driver.xhci0` is
resolvable, enumerates once, then enters the cooperative poll loop that services one request and
drains every bound endpoint each iteration. The first time any HID endpoint binds, the loop emits the
debug marker `[USB-HID-ENUM] tablet bound` exactly once (`src/orchestrator/poll/run.rs:36`); the
[debugging](debugging.md) page covers what that marker and the kernel-side input markers mean.

## Source map

Everything here is drawn from `userland/capsule_driver_usb_hid/` (the capsule source and its
`Capsule.mk`), `src/capabilities/` (the capability bits and the `can_input_source` gate),
`src/syscall/contract/cap_table/mk.rs` (the per-syscall gate), and the kernel spawn mirror under
`src/userspace/capsule_driver_usb_hid/`. Every reference above is verified against those trees.

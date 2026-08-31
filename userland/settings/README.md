# The Settings Capsule

Settings is the system control panel for NONOS: a signed userland capsule that draws its own window,
reads the keyboard and pointer, and edits the machine's policy store through capability-checked IPC. It
presents three tabs of controls (Display, Network, Security), and every row is a live field backed by the
`policy` service. Settings holds no policy of its own; it is a viewer and editor for the store, and the
store, not the capsule, decides whether any write is allowed. Its source is organized into a small set of
pillars, and this documentation mirrors that structure one page per pillar so a page can be read beside
the folder it describes.

## Identity

| Field | Value | Source |
|-------|-------|--------|
| Slug | `settings` | `userland/capsule_settings/Capsule.mk:1` |
| Service handle | `app.settings` | `Capsule.mk:2` |
| Namespace | `systems.nonos.app.settings` | `Capsule.mk:7` |
| Service endpoint | `service:4728:app.settings` | `Capsule.mk:8` |
| Reply endpoint | `reply:4729:endpoint.app.settings.reply` | `Capsule.mk:9` |
| Capability mask | `0x981d` | `Capsule.mk:13` |
| Binary name | `settings` | `Capsule.mk:5` |
| Kernel mirror | `src/userspace/capsule_settings` | `Capsule.mk:12` |

The mask decomposes into seven bits, checked against `src/capabilities/types/defs.rs`:

| Bit | Value | Grants |
|-----|-------|--------|
| CoreExec | `0x0001` | run as a process |
| Network | `0x0004` | call `net.dhcp.client` for the Wi-Fi lease status |
| IPC | `0x0008` | send and receive on its endpoints |
| Memory | `0x0010` | map its own heap and stack |
| GraphicsDisplayQuery | `0x0800` | ask the compositor for the display geometry |
| GraphicsSurfaceCreate | `0x1000` | create the window surface it draws into |
| DeviceEnum | `0x8000` | enumerate devices for the settings panels |

So `0x981d = 0x0001 + 0x0004 + 0x0008 + 0x0010 + 0x0800 + 0x1000 + 0x8000`. The kernel spawn path requests
exactly those seven capabilities and no others (`src/userspace/capsule_settings/spawn.rs:49`). `Network`
is required to read the Wi-Fi lease status from the DHCP client (`userland/capsule_settings/Capsule.mk:12`),
and `DeviceEnum` lets the panels enumerate devices. There is no FileSystem bit (`0x0040`), and no
hardware, driver, MMIO, or DMA capability. Its power to change system policy is not a capability it holds;
it comes entirely from the policy service recognising its service name, which is the whole basis of the
[policy client and write gate](policy.md) page.

## The pillars

The source under `userland/capsule_settings/src/settings/` is a set of modules declared in
`src/settings/mod.rs:17`, and the documentation is one page each. A key or click comes in through
`event`, which reads and mutates the `state` model and, on a change, sends a write through `ipc` to the
policy service; `paint` turns the current `state` into pixels. The `schema` module is the shared spine:
it lists the fields and assigns each to a tab.

```
  event/   ->   ipc/   ->   policy service
  input        the        (owns the store,
  handling     client      gates the write)
    |            |
    v            v
  state/  <->  schema/   ->   paint/
  the model    fields+tabs    the frame
```

| Page | Mirrors | What it covers |
|------|---------|----------------|
| [panels.md](panels.md) | `src/settings/schema/`, `src/settings/state/` | The three tabs and every control: field, kind, range, and what each row writes; the field list, the tab grouping, and the in-memory model behind the rows. |
| [policy.md](policy.md) | `src/settings/ipc/` | The policy client: service lookup, the read burst, the four `OP_SET` writers, the framed `call`, the trusted-setter gate on the server, and the best-effort shell toast. |
| [rendering.md](rendering.md) | `src/settings/paint/` | How a frame is produced: clear, header, tabs, the visible rows, per-kind value rendering, the scroll indicator, and the status bar. |
| [input.md](input.md) | `src/settings/event/` | Every keybinding and pointer action, the browsing and editing split, the adjust and toggle paths, the string editor filter, and the status messages. |
| [contributing.md](contributing.md) | the whole tree | Where to work, how to add a setting, the build and sign steps, and the code standards. |
| [debugging.md](debugging.md) | runtime | The boot marker, the failure modes, and where to look when hydration, a write, or the display misbehaves. |

The Wi-Fi auto-connect row lives on the Network tab and is documented with the other controls in
[panels.md](panels.md), but it is a single policy boolean and nothing more. The scanning, association, and
credential handling belong to the network capsule; for how a Wi-Fi link is brought up, see the
[networking subsystem](../../../subsystems/networking/README.md).

## Lifecycle

Settings is spawned through [verified spawn](../../../security/capsules-and-trust.md): its signature and
attestation are checked, its requested capabilities are held against its manifest ceiling, and only then
is its ELF mapped and `app.settings` registered on port 4728. A successful spawn prints `[APP-SETTINGS]
capsule spawned` on the boot log (`src/userspace/init/spawn_plan/boot.rs:26`).

1. `_start` hands `Settings::new` to the app-skeleton's `run`, which owns the surface, window, input
   subscription, and paint loop (`src/main.rs:28`).
2. `Settings::new` looks up the `policy` service and, if found, records its port and marks policy ready
   (`src/settings/app.rs:31`). The skeleton then creates a 760x520 Normal window titled `NONOS Settings`,
   subscribed to key-down input (`src/settings/manifest.rs:24`).
3. On the first event or paint after the port is known, `ensure_ready` hydrates the cache once by reading
   every field with `OP_GET`, then never runs again (`src/settings/app.rs:42`).
4. Each key-down or click flows through `on_event` to the browsing, editing, or pointer handler, which may
   send an `OP_SET` and update the cache and status on success (`src/settings/event/on_event.rs:25`).
5. `paint` projects the current `State` into the surface (`src/settings/paint/paint.rs:31`).

## Source map

Everything here is drawn from `userland/capsule_settings/` (the capsule source and its `Capsule.mk`),
`src/capabilities/types/defs.rs` (the capability bits), the kernel spawn mirror under
`src/userspace/capsule_settings/`, and the shared `userland/policy_proto/` crate (the `Field` enum,
labels, kinds, ranges, enum tables, and wire header). Every reference above is verified against those
trees.

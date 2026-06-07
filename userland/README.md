# Userland Model

This page describes how NØNOS userland is arranged: capsule source trees,
kernel mirror modules, init spawn order, and the residual supervisor loop. Read
[Architecture Overview](../architecture/overview.md) first, then read
[Syscall ABI Reference](../abi/syscalls.md).

Pages in this section:

| Page | Scope |
|------|-------|
| [SDK](sdk.md) | Syscall bindings, runtime crates, and capsule structure. |
| [Desktop](desktop.md) | GUI capsules, desktop IPC, window state, and input routing. |
| [Desktop Service Capsules](desktop-capsules.md) | Compositor, WM, input router, shell, wallpaper, image, clipboard, login, and toolkit internals. |
| [GUI Contracts](gui-contracts.md) | Window placement, move, resize, close, focus, cursor, and input delivery contracts. |
| [Protocol Atlas](protocols.md) | IPC op tables, app control frames, launch frames, and per-family protocol surfaces. |
| [Lifecycle and Launch](lifecycle.md) | Init spawn order, verified spawn, lifecycle tracking, supervisor loop, and desktop launch broker. |
| [Runtime Workflows](workflows.md) | End-to-end boot, launch, render, input, window, storage, network, and debug workflows. |
| [Capsule Inventory](capsules.md) | Complete userland capsule inventory with handles, endpoints, caps, entrypoints, and protocol refs. |
| [Applications](apps.md) | App skeleton contract, app manifests, deterministic window geometry, input masks, and direct GUI apps. |
| [Services](services.md) | Core, security, storage, desktop service, proof, policy, market, login, clipboard, image, and toolkit capsules. |
| [Core Service Capsules](core-capsules.md) | RAMFS, VFS, keyring, entropy, crypto, policy, attest, market, installer, payment, power, and proof internals. |
| [Storage Capsules](storage-capsules.md) | RAMFS, VFS, virtio block, AHCI, NVMe, and USB mass storage internals. |
| [Drivers](drivers.md) | User-mode hardware driver capsules, boot group, endpoint, capability, and protocol surface. |
| [Network Capsules](network-capsules.md) | L2, IPv4, UDP, DHCP, TCP, DNS, sockets, and Nym capsule contracts. |

This section is organized as an audit path. Start with source layout, confirm how
a capsule becomes embedded bytes, follow init into verified spawn, then move into
the desktop, app, service, driver, and network pages for subsystem detail.

---

## 1. Capsule source layout

A production capsule has two source locations. The user-mode program lives under
`userland/<capsule>`. The kernel mirror lives under `src/userspace/capsule_*`
and embeds the capsule ELF, NØNOS-ID certificate, and manifest into the kernel
build. The desktop shell mirror is the clean example: `embed.rs` includes the
ELF from `userland/capsule_desktop_shell/target/.../desktop_shell`, the
certificate from `nonos-data/trust/capsules/desktop_shell.nonos_id_cert.bin`,
and the manifest from `nonos-data/trust/capsules/desktop_shell.manifest.bin`
(`src/userspace/capsule_desktop_shell/embed.rs:17`).

The same mirror exposes one spawn function and one lifecycle state accessor.
`mod.rs` exports `spawn_desktop_shell_capsule` and `shared_state`
(`src/userspace/capsule_desktop_shell/mod.rs:17`). The spawn function builds a
`CapsuleSpecVerified` with service port, reply inbox, target triple, embedded
payloads, and requested capability bits, then calls `spawn_verified`
(`src/userspace/capsule_desktop_shell/spawn.rs:36`).

```
  +-----------------------------+
  | userland/capsule_*          |
  | no_std binary, Cargo.toml   |
  | Capsule.mk                  |
  +-------------+---------------+
                |
  +-------------+---------------+
  | build and sign              |
  +-------------+---------------+
                |
  +-----------------------------+
  | target/.../release/<bin>    |
  | nonos-data/trust/capsules   |
  | cert.bin, manifest.bin      |
  +-------------+---------------+
                |
  +-------------+---------------+
  | include_bytes               |
  +-------------+---------------+
                |
  +-----------------------------+
  | src/userspace/capsule_*     |
  | embed.rs, spawn.rs, state.rs|
  +-------------+---------------+
                |
  +-------------+---------------+
  | spawn_verified              |
  +-------------+---------------+
                |
  +-----------------------------+
  | user process on runqueue    |
  +-----------------------------+
```

## 2. Init sequence

The init capsule starts in `run_init`. Its ordered spawn sequence is RAMFS,
RAMFS smoke, core services, drivers, VFS, network, launcher registration,
desktop, market, smoke tests, then the supervisor loop
(`src/userspace/init/entry.rs:25`). The source
spells the sequence directly: `spawn_ramfs`, `spawn_core_after_ramfs`,
`spawn_drivers`, `spawn_vfs`, `spawn_network`, `launcher::register`,
`spawn_desktop`, and `spawn_market` are called before `init_loop`
(`src/userspace/init/entry.rs:28`).

The orchestrator splits those phases into small entry points. Drivers are
grouped as virtio, bus, input, NIC, USB, and storage
(`src/userspace/init/spawn_plan/orchestrator.rs:29`). Network startup calls L2,
IP, UDP, DHCP, TCP, DNS, Nym, and sockets in that order
(`src/userspace/init/spawn_plan/network.rs:17`). Desktop startup calls
input_router and compositor as GUI core, then WM, wallpaper catalog, wallpaper,
desktop shell, and desktop services (`src/userspace/init/spawn_plan/desktop_fleet.rs:17`).

```
+--------------------------+
| run_init                 |
+------------+-------------+
             |
+------------+-------------+
| ramfs and core services  |
+------------+-------------+
             |
+------------+-------------+
| driver groups            |
+------------+-------------+
             |
+------------+-------------+
| vfs and network          |
+------------+-------------+
             |
+------------+-------------+
| desktop launcher broker  |
+------------+-------------+
             |
+------------+-------------+
| desktop and market       |
+------------+-------------+
             |
+------------+-------------+
| supervisor loop          |
+--------------------------+
```

| Phase | Source |
|-------|--------|
| RAMFS first | `src/userspace/init/entry.rs:28` |
| Core after RAMFS | `src/userspace/init/spawn_plan/core.rs:22` |
| Driver groups | `src/userspace/init/spawn_plan/orchestrator.rs:29` |
| VFS | `src/userspace/init/entry.rs:32` |
| Network stack | `src/userspace/init/spawn_plan/network.rs:17` |
| Launcher broker | `src/userspace/init/entry.rs:34` |
| Desktop stack | `src/userspace/init/spawn_plan/desktop_fleet.rs:17` |
| Market | `src/userspace/init/spawn_plan/core.rs:35` |

## 3. Supervisor loop

After spawning, init lowers its own priority and yields one hundred times before
entering the supervisor loop
(`src/userspace/init/entry/lower_init_priority.rs:17`,
`src/userspace/init/entry/yield_after_spawns.rs:17`). The loop ticks the
lifecycle registry once per second, drains the launcher inbox, and then yields
(`src/userspace/init/supervisor/loop_impl.rs:25`,
`src/userspace/init/supervisor/loop_impl.rs:29`).
With the setup wizard feature enabled, the same loop waits until the setup
wizard is no longer alive, then calls `spawn_post_wizard`
(`src/userspace/init/supervisor/loop_impl.rs:35`).

The lifecycle state is capsule-local. The desktop shell mirror holds a static
`CapsuleState`, records the pid after spawn, and exposes a shared accessor
(`src/userspace/capsule_desktop_shell/state.rs:17`).

## 4. Verified spawn path

`CapsuleSpecVerified` is the kernel-side contract for a signed capsule. It holds
the service name and port, reply inbox and port, ELF bytes, certificate bytes,
manifest bytes, target triple, requested capability ceiling, and debug tag
(`src/kernel_core/process_spawn/capsule_spawn/spec.rs:31`).

Before install, preflight decodes and verifies the NØNOS-ID certificate, builds
the declared service and reply endpoints, and verifies the manifest against the
certificate, trust anchor, ELF bytes, target triple, requested caps, and
declared endpoints (`src/kernel_core/process_spawn/capsule_spawn/runner/preflight.rs:29`).
The caps installed on the process come from the verified manifest result, not
directly from `requested_caps` (`src/kernel_core/process_spawn/capsule_spawn/runner/verified.rs:23`).

Install registers the reply inbox, creates the process, registers a per-process
inbox, loads the ELF, installs caps, allocates kernel and user stacks, builds
the first user context, registers the service endpoint, and adds the pid to the
runqueue (`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:30`).

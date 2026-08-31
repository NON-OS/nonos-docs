# The Desktop Shell Capsule

`capsule_desktop_shell` is the top-level chrome of the NONOS desktop: the launcher dock at the bottom,
the menu bar and status indicators at the top, the notification toasts, the system tray, and the
spotlight panel. It is the coordination hub that ties the compositor, window manager, input router,
wallpaper, and market together, but it holds no more authority than any other graphics client. Its
source is organized into code pillars, and this documentation mirrors that structure one page per pillar
so a page can be read beside the folder it describes.

The kernel spawns it under service handle `desktop_shell` on service port 4410 with a reply inbox on
port 4411, and its capability mask is `0x100191D` (`userland/capsule_desktop_shell/Capsule.mk:20`).

## Identity

Everything the kernel and the service registry need to name and reach the shell comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Slug | `desktop-shell` | `Capsule.mk:5` |
| Service handle | `desktop_shell` | `Capsule.mk:6`, `src/userspace/capsule_desktop_shell/spawn.rs:31` |
| Namespace | `systems.nonos.desktop_shell` | `Capsule.mk:11` |
| Service endpoint | `service:4410:desktop_shell` | `Capsule.mk:12`, `spawn.rs:32` |
| Reply endpoint | `reply:4411:endpoint.desktop_shell.reply` | `Capsule.mk:13`, `spawn.rs:33`, `spawn.rs:34` |
| Capability mask | `0x100191D` | `Capsule.mk:20` |
| Binary name | `desktop_shell` | `Capsule.mk:9` |
| Kernel mirror | `src/userspace/capsule_desktop_shell` | `Capsule.mk:17` |

The committed mask `0x100191D` decomposes into eight bits, checked against `src/capabilities/types/defs.rs`:

| Bit | Value | Grants |
|---|---|---|
| CoreExec | `0x0000001` | run as a process |
| Network | `0x0000004` | call the network stack (the shell queries link status) |
| IPC | `0x0000008` | send and receive on its endpoints |
| Memory | `0x0000010` | map its own heap and stack |
| Debug | `0x0000100` | emit `MkDebug` markers (the shell-frametime counter) |
| GraphicsDisplayQuery | `0x0000800` | ask the compositor for the display geometry |
| GraphicsSurfaceCreate | `0x0001000` | create the overlay surface it draws into |
| SpawnWindow | `0x1000000` | ask the kernel to open another window instance of an app capsule |

```
  0x0000001  CoreExec               = 1
  0x0000004  Network                = 4
  0x0000008  IPC                    = 8
  0x0000010  Memory                 = 16
  0x0000100  Debug                  = 256
  0x0000800  GraphicsDisplayQuery   = 2048
  0x0001000  GraphicsSurfaceCreate  = 4096
  0x1000000  SpawnWindow            = 16777216
  ---------
  0x100191D
```

`SpawnWindow` is the bit that sets the shell apart from an ordinary app: it is the
authority to ask the kernel to open a second window instance of an embedded app
capsule, and the kernel restricts that syscall to a `SpawnWindow`-trusted capsule
(`src/syscall/contract/cap_table/mk.rs:111`). `Network` lets the shell read
link status for the tray. `Debug` is present for the shell-frametime counter; the
`Capsule.mk` comment computes the mask without it (`0x100181d`) and notes the bit
must stay in sync with `requested_caps` in the spawn mirror
(`userland/capsule_desktop_shell/Capsule.mk:17`,
`src/userspace/capsule_desktop_shell/spawn.rs`). There is no `FileSystem` bit
(64), no hardware, driver, DMA, or `GraphicsPresent` bit (16384): the shell drives
the desktop through IPC to the compositor, wm, vfs, and installer, and cannot
present a frame itself or touch a device register. It coordinates everyone, but
its own authority is bounded, and compromising it yields that mask and nothing more.

## The code pillars

The source under `userland/capsule_desktop_shell/src/` decomposes into four documented pillars. A pointer
event comes in through the served input path, may flip live `state`, which `render` turns into overlay
pixels; independently, other capsules drive the tray, toasts, and spotlight through the served
operations, and every outbound reach is a normal IPC call through a `client`.

```
  input + operations  ->   state    ->   render      ->  clients
  what comes in            what is       overlay          the compositor,
  (pointer, NDSH ops)      remembered    pixels           wm, and peers
```

| Page | Mirrors | What it covers |
|---|---|---|
| [surface.md](surface.md) | `src/render/`, `src/server/input.rs` | The user surface: the launcher dock and its nine apps, dock reveal and collapse, the menu bar, the four status indicators, notification toasts, the pointer input path, and how a frame reaches the screen. |
| [operations.md](operations.md) | `src/server/`, `src/protocol/` | The served side: the `NDSH` frame protocol, the six operations (healthcheck, tray register/update/remove, notify, spotlight), the launcher focus path, the window-manager lifecycle handling, and the loop. |
| [clients.md](clients.md) | `src/compositor_client/`, `src/wm_client/`, `src/input_router_client.rs`, `src/wallpaper_client/`, `src/market_client/`, `src/setup/` | Everything the shell reaches outward: the compositor, window manager, input router, wallpaper, market, policy, and DHCP wires, plus the setup sequence that resolves the peers and registers the overlay. |
| [state.md](state.md) | `src/state/` | The live model: the `Context`, the launcher app list, the taskbar open/pulse/visible state, the 32-slot tray table, the toast queue, the spotlight flag, and the indicator data sources. |
| [contributing.md](contributing.md) | the whole tree | Where to work, how to add a dock app or an indicator, the build and sign steps, and the code standards. |
| [debugging.md](debugging.md) | runtime | The boot marker, and where to look when the shell does not draw, an app will not launch, or the wire returns an error. |

## Lifecycle

The shell is a `no_std`/`no_main` capsule. `_start` initializes the heap, blocks in `wait_for_setup`
until every required peer is up and the overlay is registered, then runs the server loop
(`src/main.rs:37`, `src/main.rs:41`, `src/wait_for_setup.rs:19`, `src/server/runner/run.rs:29`). It
supplies its own frame protocol and its own paint routines; it is not built on the app skeleton the way
the [terminal](../terminal/README.md) is.

1. The kernel spawns the capsule through the desktop-fleet plan, which logs under the tag `DESKTOP-SHELL`
   and calls `spawn_desktop_shell_capsule` (`src/userspace/init/spawn_plan/desktop_fleet/mod.rs`). That
   path decodes the trust anchor, verifies the embedded ELF, id cert, manifest, and attestation, requests
   the eight-bit capability mask, registers `desktop_shell` on port 4410 with the reply inbox on 4411, and
   marks the capsule alive (`src/userspace/capsule_desktop_shell/spawn.rs:38`, `spawn.rs:57`).
2. `wait_for_setup` retries `setup::run` until it succeeds (`src/wait_for_setup.rs:19`). One pass resolves
   and health-checks the peers, applies the wallpaper policy, allocates the overlay, builds the `Context`,
   paints the initial chrome, registers and submits the overlay scene at z-order 1, opens the taskbar
   popup window through the wm, and subscribes to wm lifecycle and input-router events
   (`src/setup/prime/run/run.rs:21`).
3. `paint_initial` retries up to eight times to paint the chrome and land the first full-screen damage
   commit (`src/server/paint_initial.rs:24`).
4. The loop drains inbound frames, refreshes the clock and indicators once a second, re-subscribes to
   input and wm if either subscription was lost, expires toasts and taskbar pulses, and blocks on the
   display vsync (`src/server/runner/run.rs:35`).

## Source map

The whole capsule lives at `userland/capsule_desktop_shell/`. The pages above draw from `src/render/`,
`src/server/`, `src/protocol/`, `src/state/`, `src/setup/`, and the outbound clients at the crate root,
plus `Capsule.mk` for identity, `src/capabilities/types/defs.rs` for the mask bits, and the kernel spawn
mirror under `src/userspace/capsule_desktop_shell/`. Every reference above is verified against those
trees.

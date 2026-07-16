# capsule_desktop_shell

The desktop shell is the chrome: the taskbar, the system tray, notification toasts, and spotlight. It
draws that chrome into a shared overlay the compositor presents and coordinates the other desktop
services. Service `desktop_shell` on port 4410, capability mask `0x1819`. The source is
`userland/capsule_desktop_shell/`.

## The server loop

`main.rs:37` initializes the heap, waits for setup (discovering the compositor, window manager, input
router, and wallpaper, and allocating a shared overlay framebuffer), and runs the loop
(`src/server/runner/run.rs:29`):

```
  run(ctx):
      paint_initial()                    // draw the taskbar and any toasts
      loop:
          drain()                        // handle tray/notify/spotlight requests
          refresh the clock periodically
          expire toasts; refresh the taskbar on changes
          vsync_wait()
```

The frame is `NDSH` (magic `0x4E445348`), version 1.

## The operations

Six operations (`src/protocol/ops.rs`):

```
  HEALTHCHECK=1  TRAY_REGISTER=2  TRAY_UPDATE=3  TRAY_REMOVE=4  NOTIFY=5  SPOTLIGHT_OPEN=6
```

`TRAY_REGISTER` (`src/server/handlers/tray_register.rs:23`) parses a tray id and a label (up to 24
bytes), inserts it into the tray table under the caller's pid (rejecting a duplicate id for that owner),
and repaints the menu bar. `NOTIFY` enqueues a toast with a level and a body (up to 128 bytes), and
`SPOTLIGHT_OPEN` triggers the spotlight. The taskbar and tray are painted directly into the shared
overlay framebuffer, and a damage commit tells the compositor to present.

## State and honesty

The `Context` (`src/state/context.rs:19`) holds the peer ports, the tray table (owned per pid), the
taskbar state (open, active, and pulse indicators per launcher), the spotlight state, and the toast
queue. The shell is the coordination hub: it subscribes to the input router and the window manager's
lifecycle notifications, sets the wallpaper, and registers apps with the market. Honest gaps: the shell
composes its chrome by painting into the shared framebuffer rather than submitting a separate compositor
scene, there is no caller-pid check on reading tray state (any caller can enumerate the trays), and the
spotlight UX is not fully wired.

## Security analysis

The mask is `0x1819` (`CAPSULE_REQUIRED_CAPS` in `userland/capsule_desktop_shell/Capsule.mk`), decoding to
`CoreExec | IPC | Memory | GraphicsDisplayQuery | GraphicsSurfaceCreate` against `src/capabilities/types.rs`.
It has `GraphicsSurfaceCreate` because it allocates and registers a shared overlay surface during setup
(`src/setup/prime/register.rs:41`) that it paints the chrome into, and `GraphicsDisplayQuery` to size that
overlay to the display. It does not hold `GraphicsPresent`: like every other client the shell commits
damage and the [compositor](compositor.md) presents. It holds no hardware, filesystem, network, or crypto
capability, so the coordination hub of the desktop is still just a graphics client with an IPC port.

- **The shell coordinates, it does not command.** It calls peers by name (the compositor, wm, input
  router, wallpaper, and market), but it reaches each through a service lookup and a normal request, so its
  authority over them is whatever their own handlers grant a caller. It cannot, for example, present a frame
  itself or take an exclusive input grab.
- **Tray entries are owner-scoped on write.** `TRAY_REGISTER` (`src/server/handlers/tray_register.rs`) tags
  each entry with `sender_pid` and rejects a duplicate `(pid, tray_id)` with `E_BUSY`, so a capsule cannot
  register over another capsule's tray id. Labels are length-checked against `TRAY_LABEL_MAX` and the table
  is bounded (`E_NOMEM` when full).
- **Honest boundary: reads are not owner-checked.** As the state note says, there is no caller-pid check on
  reading tray state, so any caller that can reach the service can enumerate the trays. And because the
  chrome is painted straight into the shared overlay framebuffer rather than submitted as a separate
  compositor scene, the shell trusts its own paint routines to stay inside the overlay rectangle; there is
  no compositor-side clip on that overlay the way there is on a submitted layer.

## Debugging

The shell registers as `desktop_shell` on port 4410 and is reached by `mk_service_lookup("desktop_shell")`.
Its own setup depends on finding peers by name (`src/setup/discover/`): the wallpaper lookup is required and
fails setup with `"wallpaper service not announced"` if the wallpaper capsule is not up
(`src/setup/discover/require_wallpaper.rs:19`), while the market lookup is best-effort (`try_market` returns
0 when absent). So the shell's bring-up ordering is that the compositor, wm, input router, and wallpaper
register before it. The kernel spawn marker is:

```
  [SPAWN] name=desktop_shell pid=0x... caps=0x1819 entry=0x...
```

`caps=0x1819` confirms the admitted mask. The shell is also the service the [boot splash](boot-splash.md)
waits for: the splash polls `lookup("desktop_shell")` and hands off the screen once it resolves, so a splash
that never leaves the screen usually means `desktop_shell` never registered.

The runtime failure signatures are on the wire: a `TRAY_REGISTER` with a bad body length or an empty or
oversized label is `E_INVAL`, a duplicate tray id for the same owner is `E_BUSY`, and a full tray table is
`E_NOMEM`. A market call that fails is surfaced as `"market call failed"` inside the shell rather than a
protocol error to the tray caller, because the market is a best-effort peer.

## Source map

```
  userland/capsule_desktop_shell/src/server/runner/            the loop
  userland/capsule_desktop_shell/src/server/handlers/tray_register.rs   owner-scoped tray insert, E_BUSY/E_NOMEM
  userland/capsule_desktop_shell/src/server/handlers/           notify, spotlight_open, launcher, tray_update/remove
  userland/capsule_desktop_shell/src/setup/discover/           peer lookup, require_wallpaper, try_market
  userland/capsule_desktop_shell/src/setup/prime/register.rs   the shared overlay surface
  userland/capsule_desktop_shell/src/state/context.rs          tray, taskbar, toasts, peer ports
  userland/capsule_desktop_shell/Capsule.mk                    CAPSULE_REQUIRED_CAPS = 0x1819, endpoint 4410
```

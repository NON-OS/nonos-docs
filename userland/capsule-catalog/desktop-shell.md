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

## Source

```
  userland/capsule_desktop_shell/src/server/runner/     the loop
  userland/capsule_desktop_shell/src/server/handlers/    tray_register, notify, spotlight
  userland/capsule_desktop_shell/src/state/context.rs    tray, taskbar, toasts, peer ports
  userland/capsule_desktop_shell/src/setup/              peer discovery, overlay allocation
```

# Desktop and GUI Service Capsules

The desktop is a fleet of cooperating capsules: a compositor that owns the screen, a window manager, an
input router, the shell chrome, and the supporting services. Each now has a dedicated verified deep page
covering its server loop, wire protocol, per-operation logic, state, cross-capsule calls, and honest
gaps. This page indexes them.

| Capsule | Service | Caps | What it is |
|---------|---------|------|-----------|
| [compositor](compositor.md) | `compositor` :4310 | `0x7919` | Owns the screen: scene layers, damage-driven compositing, vsync present. |
| [wm](wm.md) | `wm` :4330 | `0x19` | Window lifecycle, z-order, focus, and the hit-test/focus queries the input router uses. |
| [input-router](input-router.md) | `input_router` :4320 | `0x19` | Drains the kernel input ring and routes to windows by hit-test and focus, with trusted-only grabs. |
| [desktop-shell](desktop-shell.md) | `desktop_shell` :4410 | `0x1819` | The taskbar, tray, toasts, and spotlight; coordinates the desktop services. |
| [wallpaper](wallpaper.md) | `wallpaper` :4340 | `0x1819` | The background: color or catalog image, with a fade timeline. |
| [wallpaper-catalog](wallpaper-catalog.md) | `wallpaper_catalog` | `0x19` | Serves built-in wallpaper metadata and chunked image bytes. |
| [image-codec](image-codec.md) | `image_codec` :4412 | `0x1819` | Decodes PNG/BMP/JPEG/LZ4 to ARGB with real toolkit decoders; isolated because images are untrusted. |
| [clipboard](clipboard.md) | `clipboard` :4414 | `0x19` | Bounded copy history that wipes itself on idle. |
| [login](login.md) | `login` :4416 | `0x19` | Session gate: unlocks the keyring (which is authoritative), owner-pid enforced. |
| [toolkit](toolkit.md) | `toolkit` :4610 | `0x19` | Stateless theme, animation, and component-render RPC. |
| [boot-splash](boot-splash.md) | `app.boot_splash` (client only) | `0x1819` | Boot screen that displays the kernel's attestation badge; it displays, it does not verify. Endpoint reserved but the code never binds it; it is a pure compositor client. |
| [setup-wizard](setup-wizard.md) | `app.setup_wizard` :4794 | `0x1819` | First-run config wizard that commits choices to policy. |

The pattern across the fleet: the compositor is passive and owns the frame, the window manager owns
placement and answers the input router's hit-test and focus queries, and the input router is the single
consumer of the kernel [input ring](../../subsystems/input/README.md) that fans events out. The surface
and input mechanisms are the [graphics](../../subsystems/graphics/README.md) and
[input](../../subsystems/input/README.md) subsystems, and the desktop bring-up order is the
[lifecycle](../lifecycle.md) page.

# Browser

`capsule_browser` is a signed NONOS userland GUI capsule that fetches, parses,
lays out, and paints web pages inside its own window, reaching the network only
through capability-checked IPC. It carries its own HTML parser, CSS engine, box
layout, a JavaScript engine, a TLS 1.3 client, and image and font decoders, all
compiled `no_std` into one capsule. It is built on `nonos_app_skeleton` like the
other GUI apps, so its `_start` initializes the heap and calls `run(Browser::new)`
(`userland/capsule_browser/src/main.rs:35`). Because it holds a page DOM, a box
tree, decoded rasters, and transient fetch buffers at once, it asks for a larger
heap than the shared default before the skeleton initializes (48 MiB,
`userland/capsule_browser/src/main.rs:31`); a failure there is non-fatal and the
skeleton falls back to the default.

This page is verified against the source under `userland/capsule_browser/`. The
browser is an experimental engine: it renders real pages, but it is not a
mainstream-parity browser, and the honest status of each subsystem is called out
below rather than assumed.

## Identity and capabilities

| Field | Value | Source |
|-------|-------|--------|
| Slug | `browser` | `userland/capsule_browser/Capsule.mk:1` |
| Service handle | `app.browser` | `Capsule.mk:2` |
| Namespace | `systems.nonos.app.browser` | `Capsule.mk:7` |
| Service endpoint | `service:4760:app.browser` | `Capsule.mk:8` |
| Reply endpoint | `reply:4761:endpoint.app.browser.reply` | `Capsule.mk:9` |
| Instance endpoints | `4762`/`4763`, `4764`/`4765`, `4766`/`4767` for up to three on-demand windows | `Capsule.mk:13` |
| Capability mask | `0x183d` | `Capsule.mk:18` |
| Binary name | `browser` | `Capsule.mk:5` |
| Kernel mirror | `src/userspace/capsule_browser` | `Capsule.mk:19` |
| Window | `1360x760`, Normal, title `NONOS Browser`, id `0x4252_5753` | `src/browser/manifest.rs:19`, `manifest.rs:21`, `manifest.rs:22` |

The mask `0x183d` decomposes into seven bits, checked against
`src/capabilities/types/defs.rs`:

| Bit | Value | Grants |
|-----|-------|--------|
| CoreExec | `0x0001` | run as a process |
| Network | `0x0004` | the network-capability gate its fetch path is checked against |
| IPC | `0x0008` | send and receive on its endpoints |
| Memory | `0x0010` | map its own heap and stack |
| Crypto | `0x0020` | drive the kernel crypto primitives the TLS record layer uses |
| GraphicsDisplayQuery | `0x0800` | ask the compositor for the display geometry |
| GraphicsSurfaceCreate | `0x1000` | create the window surface it paints into |

```
  0x183d = 0x0001 + 0x0004 + 0x0008 + 0x0010 + 0x0020 + 0x0800 + 0x1000
```

The `Capsule.mk:17` comment lists the mask without `Network`, but the committed
value `0x183d` includes it (`0x0004`); the comment is stale relative to the
number. The browser holds no FileSystem, driver, MMIO, IRQ, or DMA capability: it
cannot touch a device or a storage surface on its own. The fetch path opens its
transport as an IPC call to the `net.sockets` service
(`src/browser/net/call.rs:46`, via `mk_ipc_call_timeout`), and a private-by-default
path routes instead through the Nym mixnet capsule
(`src/browser/net/socket_connect_host.rs:30`, `src/browser/net/mixnet/call.rs:50`).
`Crypto` backs the in-capsule TLS 1.3 record layer (`src/browser/tls13/`); `Network`
is the capability the socket IPC is gated against.

## Engine subsystems

The source under `userland/capsule_browser/src/browser/` is one module per stage
of the pipeline. Each is a real in-tree implementation, not a shim:

| Module | Role | Status |
|--------|------|--------|
| `html/` | HTML tokenizer and tree builder | IMPLEMENTED |
| `dom/` | the document tree the parser builds and JS mutates | IMPLEMENTED |
| `css/` | CSS parsing and the cascade | IMPLEMENTED |
| `layout/` | the box model and reflow (reflows on window-width change, `src/browser/app.rs:43`) | IMPLEMENTED |
| `paint/` | turning the box tree into pixels for the compositor surface | IMPLEMENTED |
| `fonts/` | glyph rasterization | IMPLEMENTED |
| `image/` | raster image decode | IMPLEMENTED |
| `js/` | a JavaScript engine: AST, interpreter, regex, and a script world (`src/browser/js/`) | IMPLEMENTED |
| `qjs_bridge/`, `qjs_run.rs` | a QuickJS bridge (`nonos_qjs`) exposing DOM query/edit/navigation to script | IMPLEMENTED |
| `fetch/`, `http/` | the fetch pipeline: HTTP, CSS pumping, budgeted body append, `about:` pages | IMPLEMENTED |
| `tls13/` | an in-capsule TLS 1.3 client (`nonos_tls`) | IMPLEMENTED |
| `net/`, `net/mixnet/` | socket transport over `net.sockets` and the Nym mixnet path | IMPLEMENTED |
| `url/`, `proxy/`, `settings.rs`, `keymap.rs` | URL parsing, proxy config, browser settings, keymap | IMPLEMENTED |

The engine parses and paints real HTML and CSS and runs page scripts through both
the native interpreter and the QuickJS bridge (DEMONSTRATED). It is not verified
against any web-platform conformance suite, and no in-tree proof asserts
rendering correctness (NOT PROVEN). Treat site compatibility as best-effort: the
subsystems exist and run, but coverage of the full web platform is partial.

## Lifecycle

The browser is spawned through [verified spawn](../../../security/capsules-and-trust.md):
its signature, id cert, manifest, and attestation trailer are checked, its
requested capabilities are held against its manifest ceiling, and only then is its
ELF mapped. It registers `app.browser` on port 4760, creates its window surface,
subscribes for keyboard, pointer, wheel, and button input
(`src/browser/manifest.rs:23`), and enters the input-driven paint loop. The three
instance endpoints let the desktop shell open up to three additional browser
windows on demand, each registering its own service and receiving its own
compositor window (`Capsule.mk:10`).

## Source map

```
  userland/capsule_browser/Capsule.mk          identity, endpoints, capability mask
  userland/capsule_browser/src/main.rs         _start, the 48 MiB heap, run(Browser::new)
  userland/capsule_browser/src/browser/app.rs  the App impl: manifest, on_event, paint, reflow
  userland/capsule_browser/src/browser/        html, css, dom, layout, paint, js, fetch, tls13, net
  src/userspace/capsule_browser/               the kernel mirror that embeds and spawns the capsule
  src/capabilities/types/defs.rs               the capability bits behind 0x183d
```

Everything here is drawn from `userland/capsule_browser/` and its `Capsule.mk`,
and the kernel spawn mirror under `src/userspace/capsule_browser/`.

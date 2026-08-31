# Writing an app: the SDK

An application on NONOS is a capsule like everything else: a signed, attested
`no_std` binary that holds only the capabilities its manifest declared and talks
to the rest of the system through the kernel. The SDK under
`userland/sdk/` exists so you do not have to know any of that to get a window on
the screen. You write a function, you name the capabilities you want, and the
runtime does the boot handshake, the compositor wiring, and the teardown for
you.

Here is a whole application:

```rust
#![no_std]
#![no_main]
use nonos_sdk::prelude::*;

fn app() {
    let _ = App::new("Hello").window().show();
}

sdk_main!(app, caps: [WINDOW]);
```

That is `userland/sdk/examples/hello_nonos/src/main.rs`, unedited. Everything
below is an explanation of those seven lines and how far they scale.

## The entry macro is where capabilities are declared

`sdk_main!` (`userland/sdk/nonos_sdk/src/macros.rs`) is the only piece of
ceremony, and it is doing something specific. Expanded, it emits two things:

```rust
#[no_mangle] #[used] #[link_section = ".nonos.caps"]
pub static NONOS_DECLARED_CAPS: u64 = caps::BASE | caps::WINDOW;

#[no_mangle]
pub extern "C" fn _start() -> ! {
    __run(NONOS_DECLARED_CAPS, app)
}
```

The capability set is not passed as an argument or read from a config file. It
is a `u64` written into a dedicated `.nonos.caps` ELF section, so it is part of
the binary the signer signs and the kernel attests. You cannot ask for a
capability at run time that your binary did not declare at build time, because
the declaration is baked into the measured image. `caps: [WINDOW]` becomes
`caps::BASE | caps::WINDOW`; add more groups and they OR together. `BASE` is the
floor every app needs to exist and run; the named groups are the extra authority
you are asking for and will have to justify to the signer.

This is the capability model from the kernel side (see
[../../security/capabilities-and-tokens.md](../../security/capabilities-and-tokens.md))
showing up as one line of application code. A media player that never writes
`caps: [NETWORK]` cannot open a socket, and not because it politely declines:
the bit is not in its `.nonos.caps` section, so the kernel refuses the syscall.

## The runtime owns the lifecycle

`__run` is `nonos_runtime::run` (`userland/nonos_runtime/src/run.rs`), and it is
short enough to quote:

```rust
pub fn run(caps: u64, entry: fn()) -> ! {
    if boot(caps).is_err() { exit(1); }
    entry();
    run_cleanup();
    exit(0)
}
```

`boot` performs the capsule's side of the spawn handshake with the declared
caps. If it fails the process exits non-zero before your code runs at all, which
is the correct behavior: an app that could not establish its authority should
not get to pretend it did. If it succeeds your `app()` function runs, and
whenever it returns the runtime runs registered cleanup hooks and exits cleanly.
You never write `_start`, you never call the boot syscall, and you never leak a
half-initialized process on the way out.

## The App builder

`App` (`userland/sdk/nonos_app/src/`) is a small builder, one method per file in
the crate's house style:

- `App::new(title)` starts one.
- `.window()` asks for a window (sets `want_window`).
- `.size(w, h)` sets its dimensions.
- `.background(color)` sets the fill.
- `.show()` presents a bare window and returns an `i64` status.
- `.run(root)` presents the window and enters the event loop with a root
  control, and never returns (`-> !`).

`.show()` is for the trivial case. `.run()` is for a real app, because it takes
something that implements `Control` and drives it: paint, input, repeat.

## The widget layer

`nonos_ui` (`userland/sdk/nonos_ui/`) is the drawing and interaction vocabulary:
`Canvas` to draw into, `Color`, `Rect`, the `Widget` type, and the `Control`
trait that `App::run` expects. A `Control` is anything that can lay itself out in
a rectangle, paint onto a canvas, and respond to input events.

`nonos_appkit` (`userland/sdk/nonos_appkit/`) is the batteries-included set built
on top of it: `Button`, `Label`, `Panel`, and a `Theme` so you are not choosing
colors by hand. The second example app,
`userland/sdk/examples/appkit_demo/src/main.rs`, is the whole thing in practice:

```rust
let theme = Theme::dark();
let root = Panel::new()
    .label(Label::new(Rect { x: 16, y: 16, w: 300, h: 8 }, "NONOS App Kit", theme.foreground))
    .button(Button::new(Rect { x: 16, y: 40, w: 120, h: 28 }, "Launch",
                        theme.button_bg, theme.button_fg, theme.accent));
App::new("App Kit Demo").size(480, 320).background(theme.background).run(root);
```

A `Panel` is a `Control` that owns child controls, so a real interface is a tree
of these handed to `App::run`. Text rendering underneath comes from `nonos_font`
(`glyph_for_ascii`, `text_width`, `GLYPH_WIDTH`/`GLYPH_HEIGHT`), a fixed-cell
bitmap font, which is why layout math in the examples is done in exact pixels.

## Going below the widgets

Most apps never need this, but the SDK does not hide the machine. `nonos_desktop`
(`userland/sdk/nonos_desktop/`) exposes the compositor and window-manager
protocol directly: `scene_submit` / `scene_remove` / `damage_commit` to drive
the compositor, `window_open` / `window_close` / `window_focus` for the window
manager, `drain_input` and the input-router `subscribe` for events, and
`lookup_peers` to find other capsules. `App::run` is built on exactly these
calls; if you need to do something the builder does not express, you drop to this
layer and talk to the compositor capsule yourself. `nonos_window`
(`userland/sdk/nonos_window/`) is the `Window` type that ties a surface to a
window-manager handle.

## The prelude

`use nonos_sdk::prelude::*;` (`userland/sdk/nonos_sdk/src/prelude.rs`) is the one
import a normal app needs. It pulls in the `sdk_main!` macro and, through
`nonos_prelude`, the everyday surface: `App`, `Window`, the `nonos_ui` types
(`Canvas`, `Color`, `Control`, `Rect`, `Widget`), and the runtime's `exit`,
`log`, and `yield_now`. Anything beyond that (`nonos_appkit`, `nonos_desktop`)
you import explicitly, which keeps the intent of an app visible in its imports.

## From source to a running capsule

Writing the code is half of it. The binary still has to be built for the NONOS
user target, have its manifest and certificate produced, be signed, and be
installed. That pipeline (the `Capsule.mk` contract, `capsule-sign`, and the
install step) is documented in [../sdk.md](../sdk.md) and
[../../build/signing.md](../../build/signing.md). The short version: the same
`.nonos.caps` value the macro emitted is what the manifest declares and the
kernel checks at spawn, so the capability story is consistent from the line of
Rust you wrote to the moment the process is scheduled.

## Status

| Piece | Source | Status |
|---|---|---|
| `sdk_main!` capability declaration into `.nonos.caps` | `nonos_sdk/src/macros.rs` | IMPLEMENTED; the bits are what the kernel attests |
| Runtime lifecycle (`boot` / entry / cleanup / exit) | `nonos_runtime/src/run.rs` | IMPLEMENTED |
| `App` builder, `show`, `run` | `nonos_app/src/` | IMPLEMENTED |
| Widget layer (`Canvas`, `Control`, `Widget`) | `nonos_ui/src/` | IMPLEMENTED |
| App kit (`Button`, `Label`, `Panel`, `Theme`) | `nonos_appkit/src/` | IMPLEMENTED; DEMONSTRATED by the example apps |
| Compositor / WM / input access | `nonos_desktop/src/` | IMPLEMENTED |
| Bitmap font | `nonos_font/src/` | IMPLEMENTED |

Both example apps under `userland/sdk/examples/` build and run; they are the
DEMONSTRATED path for the SDK.

## Source

`userland/sdk/`. Read `nonos_sdk/src/macros.rs` first to see how an app is
declared, then `nonos_runtime/src/run.rs` for the lifecycle, `nonos_app/src/` for
the builder, and `nonos_ui/` and `nonos_appkit/` for the drawing and widget
layers. `examples/hello_nonos` and `examples/appkit_demo` are the two ends of the
range.

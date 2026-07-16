# toolkit

The toolkit is the shared GUI service: a theme engine, an animation clock, and a component renderer that
capsule GUIs call so they share a consistent look and coordinated animation. It is a stateless RPC leaf
service. Service `toolkit` on port 4610, capability mask `0x19`. The source is `userland/toolkit/` (a
binary over the `nonos_toolkit` library).

## The server loop

`main.rs:26` initializes the heap (tolerating an already-initialized heap) and runs the loop
(`src/server/runner.rs:29`): receive on port 4610, decode the header, dispatch, encode the reply. The
frame is `NOTK` (magic `0x4E4F544B`) with a 16-byte header carrying the op, a status, a request id, and
a payload length.

## The operations

Five operations (`src/protocol/ops.rs:19`):

```
  HEALTHCHECK=0  THEME_APPLY=1  ANIMATION_TICK=2  COMPONENT_RENDER=3  THEME_GET=4
```

`dispatch` (`src/server/dispatch.rs:25`) routes each: `THEME_APPLY` sets the global theme, `THEME_GET`
returns a 24-byte snapshot (five ARGB colors, background, surface, accent, text, border, plus a revision
counter), `ANIMATION_TICK` advances the animation store by a nanosecond delta and returns the new state,
and `COMPONENT_RENDER` renders a named component (button, checkbox, and so on) from its parameters. An
unknown op is `E_BAD_OP`.

## State and honesty

The toolkit holds a global theme (colors plus a revision) and an animation store, and it is otherwise
stateless per request and calls no other service, it is a leaf. Honest gaps stated from the code: there
is no caller authentication (any capsule can apply a theme or tick animations), the animation store is
global so concurrent ticks from different callers can race, `THEME_APPLY` does not validate its payload,
and the revision is a wrapping 32-bit counter.

## Security analysis

The mask is `0x19` (`CAPSULE_REQUIRED_CAPS` in `userland/toolkit/Capsule.mk`), decoding to `CoreExec | IPC | Memory` against `src/capabilities/types.rs`. Despite being the GUI toolkit, it holds no graphics capabilities at all: it does not create surfaces, does not query the display, and does not present. It computes and returns bytes, colors, animation state, and rendered component pixels in a caller-supplied reply buffer, and the caller is the one that owns a surface and paints into it. That is the right shape for a leaf: the toolkit is pure computation over IPC, so a compromise of the toolkit cannot draw on the screen or reach a device on its own.

- **No graphics reach.** The absence of `GraphicsSurfaceCreate`, `GraphicsSurfaceMap`, and `GraphicsPresent` means `COMPONENT_RENDER` cannot blit anywhere itself; it fills the reply buffer and the calling capsule composites it. The toolkit never becomes a second painter competing with the [compositor](compositor.md).
- **Passive leaf.** `dispatch` (`src/server/dispatch.rs:25`) calls no other service; every op is answered from the global theme and animation store the capsule holds. Its attack surface is the five fixed ops of the `NOTK` protocol.
- **Honest boundary: no caller authentication.** Any capsule that can reach port 4610 can `THEME_APPLY` and change the global theme every GUI reads, and `THEME_APPLY` does not validate its payload (stated in the honesty note above). The global animation store means concurrent ticks from different callers race on shared state. The trust model here is that the theme is cosmetic: a bad theme is ugly, not a boundary crossing, because the toolkit cannot itself put those pixels on screen.

## Debugging

The toolkit registers as `toolkit` on port 4610 and receives there (`src/server/runner.rs:29`). A client reaches it by `mk_service_lookup("toolkit")` and sending to the resolved port; a zero port or pid from that lookup means the capsule never came up. The kernel spawn marker is:

```
  [SPAWN] name=toolkit pid=0x... caps=0x19 entry=0x...
```

`caps=0x19` confirms it was admitted with exactly `CoreExec | IPC | Memory`. `toolkit` is one of the names the spawn tracer prints install-stage lines for (`src/kernel_core/process_spawn/capsule_spawn/runner/install/trace.rs:20`), so a stall in its install is visible rather than silent.

The failure signatures are on the wire: an unknown op is `E_BAD_OP` (`src/server/dispatch.rs`), a `THEME_GET` whose reply buffer is smaller than the 24-byte snapshot is `E_BAD_OP` as well (the `theme_get` length guard), and a rendered component that does not fit the caller's buffer comes back short. Because the store is global, a theme that looks wrong across every app at once points at a stray `THEME_APPLY`, not at one client.

## Source map

```
  userland/toolkit/src/server/runner.rs      the port-4610 loop
  userland/toolkit/src/server/dispatch.rs     op routing, E_BAD_OP, theme_get length guard
  userland/toolkit/src/protocol/              the NOTK header, ops, THEME_PAYLOAD_LEN
  userland/toolkit/Capsule.mk                 CAPSULE_REQUIRED_CAPS = 0x19, endpoint 4610
  (library) nonos_toolkit                     the theme, animation, and component implementations
```

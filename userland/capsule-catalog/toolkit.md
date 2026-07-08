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

## Source

```
  userland/toolkit/src/server/runner.rs     the loop
  userland/toolkit/src/server/dispatch.rs    op routing
  userland/toolkit/src/protocol/             the NOTK header and ops
  (library) nonos_toolkit                    the theme, animation, and component implementations
```

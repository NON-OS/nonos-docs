# capsule_snake (full reference)

`capsule_snake` is the classic snake game as a signed NONOS capsule: a normal GUI window with a real
game loop behind it. It is also the smallest complete interactive application in the tree, so it doubles
as a worked example of a self-contained app that owns nothing but its own state and its surface. Where
[capsule_terminal](terminal/README.md) shows how large an [app-skeleton](../writing-an-app.md) app can get,
snake shows how small one can be and still be a real, verified, least-privilege capsule.

The kernel spawns it under service handle `app.snake` on service port 4732 with a reply port on 4733, and
its capability mask is `0x1819` (`userland/capsule_snake/Capsule.mk:11`). The source is
`userland/capsule_snake/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Controls and interaction](#controls-and-interaction)
- [Game rules and scoring](#game-rules-and-scoring)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

Snake is an ordinary NONOS GUI application built on the app skeleton. Its entry point hands its `App`
implementation to the skeleton's `run`, so the runtime owns the surface, the window, the input
subscription, and the paint loop, and the capsule supplies four things: a manifest for a normal window,
an `on_event` that turns keystrokes into steering and control actions, a `paint` that draws the frame,
and the pair `on_tick`/`tick_interval_ms` that advances the game one cell on a self-timed cadence
(`userland/capsule_snake/src/main.rs:28`, `src/snake/app.rs:35`).

The whole game is a small owned state machine. The snake is a vector of `(col, row)` cells on a fixed
28x18 grid, the food is one cell, and a phase enum drives everything else: `Ready`, `Running`, `Paused`,
`GameOver` (`src/snake/state.rs:47`, `src/snake/grid.rs:18`). There are no globals and no shared mutable
state; each `SnakeApp` holds one `Game` (`src/snake/app.rs:25`). Randomness for food placement is a local
xorshift seeded once from the wall clock, so the capsule needs no entropy service and no syscall beyond
the millisecond timer it already reads to seed and to pace ticks (`src/snake/rng.rs:19`,
`src/snake/state.rs:76`).

## Identity

Everything the kernel and the service registry need to name and reach the game comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `snake` | `Capsule.mk:1` |
| Service handle | `app.snake` | `Capsule.mk:2`, `src/userspace/capsule_snake/spawn.rs:30` |
| Service port | `4732` | `Capsule.mk:8`, `spawn.rs:31` |
| Namespace | `systems.nonos.app.snake` | `Capsule.mk:7` |
| Service endpoint | `service:4732:app.snake` | `Capsule.mk:8` |
| Reply endpoint | `reply:4733:endpoint.app.snake.reply` | `Capsule.mk:9`, `spawn.rs:32`, `spawn.rs:33` |
| Cargo feature | `nonos-capsule-snake` | `Capsule.mk:6` |
| Capability mask | `0x1819` | `Capsule.mk:11` |
| Binary name | `snake` | `Capsule.mk:5`, `Cargo.toml:16` |
| Kernel mirror | `src/userspace/capsule_snake` | `Capsule.mk:12` |

The mask `0x1819` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x0001  CoreExec                bit()  1     types.rs:56
  0x0008  IPC                     bit()  8     types.rs:59
  0x0010  Memory                  bit() 16     types.rs:60
  0x0800  GraphicsDisplayQuery    bit() 2048   types.rs:67
  0x1000  GraphicsSurfaceCreate   bit() 4096   types.rs:68
  ------
  0x1819  = 1 + 8 + 16 + 2048 + 4096
```

The kernel spawn path requests exactly those five capabilities and no others
(`src/userspace/capsule_snake/spawn.rs:49`). There is no `Network` bit (4), no `FileSystem` bit (64), and
no hardware, driver, MMIO, IRQ, or DMA capability in the mask. This is the same envelope the terminal
holds, and for the same reason: the game can create a surface, learn how big it is, and speak IPC, and
that is all it can do.

## Controls and interaction

Input arrives as events on the manifest's input subscription. The window requests three input kinds:
key-down, absolute pointer, and button-down (`src/snake/manifest.rs:23`), but the game acts only on
key-down. `on_event` returns immediately for anything that is not a key-down, so pointer motion and
clicks are ignored by the game logic (the skeleton still uses them for window dragging and the title-bar
buttons) (`src/snake/input.rs:24`).

Every control is dispatched in `on_event` and its two helpers. This is the complete set; there is nothing
else.

Steering (each direction accepts the arrow key and both cases of its WASD letter):

| Input | Key code | Action | Source |
|---|---|---|---|
| Up arrow / `w` / `W` | `KEY_UP` (0x1201), 0x77, 0x57 | steer up | `src/snake/input.rs:39` |
| Down arrow / `s` / `S` | `KEY_DOWN` (0x1202), 0x73, 0x53 | steer down | `src/snake/input.rs:40` |
| Left arrow / `a` / `A` | `KEY_LEFT` (0x1203), 0x61, 0x41 | steer left | `src/snake/input.rs:41` |
| Right arrow / `d` / `D` | `KEY_RIGHT` (0x1204), 0x64, 0x44 | steer right | `src/snake/input.rs:42` |

The `KEY_UP`..`KEY_RIGHT` codes are the skeleton's navigation constants, mirrored from the PS/2 driver's
keycode table (`userland/app_skeleton/src/input/keys.rs:24`).

Game controls:

| Input | Key code | Action | Source |
|---|---|---|---|
| Enter | `KEY_ENTER` (0x0D) | restart, only from `GameOver` | `src/snake/input.rs:31`, `input.rs:66` |
| Space / `p` / `P` | 0x20, 0x70, 0x50 | toggle pause (`Running` <-> `Paused`) | `src/snake/input.rs:32`, `input.rs:74` |

Steering behaviour, in `steer` (`src/snake/input.rs:47`):

- A reversal is rejected. A direction equal to the opposite of the current direction is ignored, so the
  snake cannot turn back into its own neck (`src/snake/input.rs:48`, `src/snake/state.rs:37`).
- From `Ready`, the first accepted direction key sets the direction and flips the phase to `Running`,
  which is what starts the game (`src/snake/input.rs:52`).
- While `Running`, a direction key sets `pending`, the direction that the next tick will adopt; it does
  not turn the snake mid-cell (`src/snake/input.rs:59`). Buffering exactly one turn this way means a fast
  double tap cannot fold the snake back on itself between two ticks.
- While `Paused` or after `GameOver`, direction keys do nothing (`src/snake/input.rs:62`).

Restart is gated. Enter is honoured only in `GameOver`; in any other phase it is ignored, so a stray
Enter mid-run cannot wipe the board (`src/snake/input.rs:67`). Restart calls `Game::reset`, which recenters
a length-3 snake, clears the score, restores the starting speed, replaces the food, and returns to
`Ready` (`src/snake/state.rs:82`).

Pause toggles only between `Running` and `Paused`; from `Ready` or `GameOver` the toggle is a no-op that
leaves the phase where it was (`src/snake/input.rs:74`). Pause is cooperative with the tick: the tick does
nothing unless the phase is `Running` (see below), so a paused game consumes no motion.

Every accepted control returns `EventOutcome::Repaint` so the frame reflects the change immediately; a
rejected or no-op input returns `EventOutcome::Idle` and the frame is left alone
(`userland/app_skeleton/src/app/event_outcome.rs:18`).

## Game rules and scoring

The game advances one cell per tick, and the tick is self-timed. `App::on_tick` calls `step`, and
`App::tick_interval_ms` reports the current interval, which the skeleton's run loop uses to decide when to
call `on_tick` again (`src/snake/app.rs:45`, `userland/app_skeleton/src/runner/entry.rs:55`). Because the
interval is read from game state on every loop, a speed change takes effect on the very next tick.

A tick does nothing unless the phase is `Running`; in `Ready`, `Paused`, or `GameOver` it returns `false`
so the skeleton skips the repaint (`src/snake/step.rs:22`). When it does run, `step`:

1. Adopts the buffered `pending` direction (`src/snake/step.rs:25`).
2. Computes the next head cell one step in that direction (`src/snake/step.rs:27`).
3. Ends the game if the next cell is off the 28x18 board or lands on the snake's own body. Self-collision
   is checked against the body minus its tail cell, because the tail is about to move out of the way on a
   non-eating step (`src/snake/step.rs:33`). On a collision the phase becomes `GameOver` and the tick
   returns (`src/snake/step.rs:35`).
4. Otherwise inserts the new head. If the new head is on the food, the score increases by one, the
   interval drops by `SPEEDUP_MS` (down to the floor), and new food is placed; if not, the tail cell is
   popped so the length stays the same (`src/snake/step.rs:39`).

Timing and scoring constants (`src/snake/state.rs:24`):

| Constant | Value | Meaning |
|---|---|---|
| `START_INTERVAL_MS` | 160 | tick interval at the start of a game |
| `MIN_INTERVAL_MS` | 80 | fastest the game will ever tick |
| `SPEEDUP_MS` | 4 | milliseconds shaved off the interval per food eaten |

So each food eaten adds one to the score and speeds the game by 4 ms, and the interval is clamped so it
never falls below 80 ms no matter how long the snake gets (`src/snake/step.rs:42`). Score is a plain
`u32` counter starting at zero and reset by `reset`; there is no persisted high score, so it does not
survive a restart or a respawn (`src/snake/state.rs:60`, `src/snake/state.rs:90`).

Food placement is a bounded-retry pick over the grid: up to 64 xorshift draws for a cell not on the
snake, falling back to the first free cell by scan if all 64 draws collided, which keeps placement finite
even when the board is nearly full (`src/snake/rng.rs:28`, `rng.rs:39`). The generator is a local
xorshift64 seeded once from `mk_time_millis` with the low bit forced set so the seed is never zero
(`src/snake/rng.rs:19`, `src/snake/state.rs:76`).

The board renders in four phases. `paint` clears the background, draws the header and the board, then
overlays a banner for whichever non-running phase is active (`src/snake/paint/mod.rs:28`):

- `Ready` shows `PRESS A DIRECTION KEY` (`src/snake/paint/overlay.rs:25`).
- `Paused` shows `PAUSED` (`src/snake/paint/overlay.rs:30`).
- `GameOver` shows `GAME OVER` and `ENTER TO RESTART` (`src/snake/paint/overlay.rs:35`).
- `Running` draws no overlay (`src/snake/paint/mod.rs:37`).

The header draws the `SCORE` label and the live score value (`src/snake/paint/header.rs:26`); the board
draws the food, then the body segments, then the head last so the head sits on top
(`src/snake/paint/board.rs:27`). The board layout is recomputed from the surface size each frame and
centered, so the cells scale to fit whatever window size the compositor grants
(`src/snake/paint/layout.rs:33`).

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` calls the skeleton's `run(SnakeApp::new)`, and the skeleton
owns everything from there (`src/main.rs:27`). The `snake` module has one file per concern: `app` (the
`App` impl and the `Game` wrapper), `state` (the `Game` struct, `Dir`, `Phase`, and the constants),
`input` (the event handlers), `step` (one tick of motion), `grid` (the board dimensions and window
geometry), `rng` (xorshift and food placement), `manifest` (the window request), and `paint` (the
renderer, itself split into `layout`, `header`, `board`, and `overlay`) (`src/snake/mod.rs:17`).

The model is a single `Game`: the body vector, the current and pending directions, the food cell, the
score, the phase, the current tick interval, and the RNG state (`src/snake/state.rs:55`). `SnakeApp` wraps
exactly one `Game` and forwards the four `App` methods to the free functions in the submodules
(`src/snake/app.rs:35`).

Lifecycle:

1. The kernel spawns the capsule at boot through the desktop-fleet spawn plan, which calls
   `spawn_snake` under the `nonos-capsule-snake` feature (`src/userspace/init/spawn_plan/apps.rs:22`,
   `apps.rs:111`). That path verifies the embedded ELF, id cert, manifest, and attestation trailer,
   registers `app.snake` on port 4732, and on success logs `[APP-SNAKE] capsule spawned`
   (`src/userspace/capsule_snake/spawn.rs:36`, `src/userspace/init/capsule_boot/run.rs:29`,
   `src/sys/boot_log/output.rs:33`).
2. The skeleton `run` waits for a delivery, builds the `SnakeApp`, opens the window from the manifest (a
   `Normal` window titled `Snake`, sized from the grid, subscribing to key-down, pointer, and button
   input), primes the first frame, and enters the event-and-tick loop
   (`userland/app_skeleton/src/runner/entry.rs:31`, `runner/boot.rs:39`, `src/snake/manifest.rs:28`).
3. Each key-down flows into `on_event`; an accepted control returns `Repaint` and the frame is redrawn
   (`src/snake/input.rs:23`).
4. On the tick cadence the loop calls `on_tick`; if it returns `true` and the window is not minimized the
   frame is repainted, so the snake only forces a redraw on frames where it actually moved or died
   (`userland/app_skeleton/src/runner/entry.rs:55`).
5. `paint` projects the `Game` into the surface each frame, and the compositor presents it
   (`src/snake/paint/mod.rs:28`).

## Protocol and IPC

Snake defines no application opcodes of its own. The `app.snake` service on port 4732 and the reply inbox
on 4733 are registered for it by the spawn record, and the app never handles an inbound request frame; it
only produces frames. Everything it does that reaches outside the capsule is one of the calls the app
skeleton makes on its behalf.

Surface and window, through the skeleton's setup path, which is why the mask needs
`GraphicsSurfaceCreate`: `open_window` allocates the backing buffer, registers and shares it as a surface
handle, and announces the window to the compositor and window manager
(`userland/app_skeleton/src/setup/open.rs:26`). The skeleton resolves those peers by name at startup:
`compositor`, `wm`, and `input_router` (`userland/app_skeleton/src/discover/require.rs:31`,
`require.rs:34`, `require.rs:37`). The frame the game paints lands in that shared surface and the
compositor presents it.

Input subscription, to `input_router`: the skeleton subscribes the window to the manifest's
`input_kind_mask`, which for snake is key-down, absolute pointer, and button-down
(`userland/app_skeleton/src/runner/boot.rs:46`, `src/snake/manifest.rs:26`). The game consumes only
key-down; the pointer and button kinds are used by the skeleton for window dragging and the title-bar
controls, not by the game.

Timer: the only libc syscall the game itself makes is `mk_time_millis`, used once to seed the RNG and
never again during play (`src/snake/state.rs:76`, `userland/libc/src/time/wall.rs:19`). The skeleton reads
the same clock in its run loop to pace ticks, but that is the runtime's call, not the game's
(`userland/app_skeleton/src/runner/entry.rs:54`).

Notably, the game makes no display-size query at runtime. It sizes its window from fixed grid constants in
the manifest and recomputes the board layout from the `PaintBuffer`'s own `width`/`height` each frame, so
`GraphicsDisplayQuery` is present in the standard app envelope but the game logic does not depend on it
(`src/snake/manifest.rs:28`, `src/snake/paint/layout.rs:33`). There is no VFS call, no clipboard call, no
network call, and no installer call anywhere in the capsule.

## Security analysis

Snake is the cleanest least-privilege example in the catalog. Its mask `0x1819` grants CoreExec, IPC,
Memory, GraphicsDisplayQuery, and GraphicsSurfaceCreate, and nothing else
(`Capsule.mk:11`, `src/userspace/capsule_snake/spawn.rs:49`). There is no Network bit, no FileSystem bit,
and no Hardware, Driver, MMIO, IRQ, or DMA capability. The game cannot open a file, cannot open a socket,
cannot touch a device register, and cannot enumerate hardware. It can create one surface, ask the display
its size, and speak IPC to the compositor, window manager, and input router that the app skeleton already
talks to on every app's behalf.

The attack surface reduces to this: the capsule receives input events and produces pixels. It reads no
external input other than key, pointer, and button events delivered by the input router, and it writes
nothing but its own surface. There is no parser for untrusted file or network data, because there is no
file or network path. The one source of nondeterminism, food placement, is a local xorshift with a bounded
retry loop and a deterministic fallback, so it cannot loop unboundedly or index out of range
(`src/snake/rng.rs:28`). The grid is fixed-size and every board index is derived from cells the step logic
already bounded against the wall, so a body that fills the board still places food through the scan
fallback rather than spinning (`src/snake/rng.rs:39`).

The code holds to the kernel's no-panic rule end to end. Motion arithmetic is on `i16` cell coordinates
checked against the board before use (`src/snake/step.rs:33`); the score is a `u32` counter with no
subtraction; the interval uses `saturating`-style `max` against its floor
(`src/snake/step.rs:42`); and food placement never unwraps. The release profile is `panic = "abort"`, so
even a reached panic is a clean abort rather than unwinding into the runtime (`Cargo.toml:24`). Isolation
from other capsules is the kernel's, not the game's: snake is a CPL 3 user binary that is verified and
enrolled at spawn like every other capsule and can only speak IPC and paint its own surface.

## How to contribute

The source lives at `userland/capsule_snake/`. The game logic is small enough to hold in your head:
`src/snake/state.rs` is the model and the tunables, `src/snake/step.rs` is one tick, `src/snake/input.rs`
is every control, and `src/snake/paint/` is the renderer.

Common changes and where they go:

1. Board size or window geometry: the grid constants in `src/snake/grid.rs:17` (`CELL`, `COLS`, `ROWS`,
   the margins, and the derived `WIN_W`/`WIN_H`). Everything downstream, the layout math and the paint
   code, reads these, so changing `COLS`/`ROWS` resizes the game with no other edit.
2. Speed and scoring: the three constants in `src/snake/state.rs:24` (`START_INTERVAL_MS`,
   `MIN_INTERVAL_MS`, `SPEEDUP_MS`). `step` applies them; no other file needs touching
   (`src/snake/step.rs:42`).
3. A new control: add the key code to the match in `on_event`, and if it needs new behaviour add a small
   helper next to `steer`, `restart`, and `toggle_pause` (`src/snake/input.rs:30`). Return `Repaint` for
   a change the player should see and `Idle` for a no-op.
4. Game rules: the collision and growth logic is all in `step` (`src/snake/step.rs:21`). Keep it panic
   free: bound every new index against `COLS`/`ROWS` before you use it, the way the wall check does.
5. Appearance: the colours and text are constants at the top of the `paint` submodules
   (`src/snake/paint/board.rs:22`, `src/snake/paint/overlay.rs:21`, `src/snake/paint/header.rs:23`).

To build and sign the capsule, use the generated per-slug make targets that `include nonos-mk/capsule.mk`
produces for every capsule (`userland/capsule_snake/Capsule.mk:14`, `nonos-mk/capsule.mk:158`):

```
  make nonos-mk-snake                build the capsule ELF
  make nonos-mk-snake-sign           produce the id cert, manifest, and attestation trailer
  make nonos-mk-snake-verify         verify the signed artifacts against the trust anchor
  make nonos-mk-check-snake-keys     check the per-capsule signing keys exist
```

For a running desktop that includes the game, `make nonos-mk-snake-prod` builds the full desktop GUI image
with snake in the fleet (`Makefile:1164`).

Code standards the capsule must meet, and already does: `cargo fmt` and a clean `cargo clippy`; no
panics, `unwrap`, or `expect` anywhere in the capsule (every branch is total, and the release profile is
`panic = "abort"`, `Cargo.toml:24`); modular files, one unit per file, with `mod.rs` used only for module
wiring and re-exports (`src/snake/mod.rs`); and the AGPL header at the top of every source file, matching
the header every existing module carries.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints `[APP-SNAKE]
capsule spawned` (tag `APP-SNAKE`, message `capsule spawned`) from the boot log
(`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line means the
capsule never started, usually a signature, manifest, or capability failure; the error path prints an
`[ERROR]` line built from the spawn error instead (`src/userspace/init/capsule_boot/run.rs:32`). Snake is
compiled into the fleet only under the `nonos-capsule-snake` feature, so on a build without that feature
`spawn_snake` is the empty stub and no line appears at all (`src/userspace/init/spawn_plan/apps.rs:115`).

Failure modes and where to look:

- Window opens but keys do nothing. The game acts only on key-down and ignores every other event kind
  (`src/snake/input.rs:24`). If the arrows do nothing, the suspect is the input path into the app
  (compositor, wm, input_router), which the skeleton resolves by name at startup and which the game does
  not control (`userland/app_skeleton/src/discover/require.rs:31`).
- The snake will not turn. A key equal to the opposite of the current direction is dropped on purpose, and
  a turn only applies on the next tick, so a rapid reverse looks like a dead key but is the anti-reversal
  guard working (`src/snake/input.rs:48`, `src/snake/step.rs:25`).
- Enter does nothing. Restart is honoured only from `GameOver`; in any other phase Enter is a deliberate
  no-op (`src/snake/input.rs:67`).
- The game never starts. From `Ready` nothing moves until the first direction key flips the phase to
  `Running`; Space and Enter alone will not start it (`src/snake/input.rs:52`).
- Motion feels frozen. A paused game and a game-over game both return `false` from the tick and paint no
  overlay change, so the board looks static; the overlay text (`PAUSED`, `GAME OVER`) tells the two apart
  from `Running` (`src/snake/step.rs:22`, `src/snake/paint/overlay.rs:30`).

There is no serial self-test build and no debug marker beyond the spawn line; the game is small enough to
reason about from the four logic files directly.

## Source map

```
  src/main.rs                       _start -> run(SnakeApp::new)
  src/snake/mod.rs                  module wiring, re-exports SnakeApp
  src/snake/app.rs                  SnakeApp: the App impl, forwards manifest/on_event/paint/on_tick
  src/snake/state.rs                Game, Dir, Phase, and the speed/score constants
  src/snake/input.rs                every control: steering, restart, pause, anti-reversal
  src/snake/step.rs                 one tick of motion: move, collide, eat, speed up
  src/snake/grid.rs                 board dimensions (28x18) and window geometry
  src/snake/rng.rs                  xorshift64 and bounded-retry food placement
  src/snake/manifest.rs             the window request (title, size, input mask)
  src/snake/paint/                  the renderer (layout, header, board, overlay)
  Capsule.mk                        slug, handle, ports, capability mask, kernel mirror
  Cargo.toml                        crate, panic=abort, AGPL license
  src/userspace/capsule_snake/      the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/apps.rs   the desktop-fleet spawn entry
  userland/app_skeleton/src/runner/       the run loop that drives on_event/on_tick/paint
  userland/app_skeleton/src/setup/        the surface and window open path
  nonos-mk/capsule.mk               the generated nonos-mk-snake[-sign|-verify] targets
```

Every reference above is verified against those trees.

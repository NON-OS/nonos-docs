# Debugging capsule_calculator

This page lists the one log marker the calculator's boot path emits, explains why the capsule itself is
silent, and walks the concrete failure modes with where to look for each. For the model see the
[README](README.md), the [input model](input.md), the [engine](engine.md), and the
[rendering](rendering.md) pages in this folder.

## The one boot marker

The calculator holds no `Debug` capability and emits no serial markers of its own, so the only log evidence
is the kernel-side boot line. On a successful boot the kernel logs `[APP-CALCULATOR] capsule spawned` from
the capsule boot path: the `Ok` arm calls `boot_log::ok(prefix, "capsule spawned")` with the prefix
`APP-CALCULATOR` supplied by the spawn plan (`src/userspace/init/capsule_boot/run.rs:29`,
`src/userspace/init/spawn_plan/apps.rs:62`). If that line is absent the capsule never started, and the
`Err` arm logged an error line through `boot_log::error` instead (`capsule_boot/run.rs:32`), which is the
usual signature, manifest, or capability failure surfaced by the verified spawn
(`src/userspace/capsule_calculator/spawn.rs:37`).

Why the capsule is otherwise silent: the spawn spec sets an empty `debug_tag` and the mask omits the
`Debug` bit (256), so the kernel grants the capsule no serial surface and it can emit nothing on the wire
(`src/userspace/capsule_calculator/spawn.rs:55`, `Capsule.mk:11`). Debugging the running capsule is
therefore done through its window behaviour, not through logs it cannot write.

## Failure modes

### Window opens but nothing responds

The window subscribes to key-down, pointer button-down, and absolute pointer input
(`src/calc/manifest.rs:23`), and `on_event` drops any other event kind as `Idle`
(`src/calc/event/router.rs:27`). If keys and clicks both do nothing, the app never sees them, so the
suspect is the input path into the app (compositor, wm, input_router), not the calculator.

### A key does nothing

The classifier ignores every code above `0x7F` and every ASCII byte not in its map
(`src/calc/event/key_classifier.rs:30`, `key_classifier.rs:52`), so an unmapped key is expected to be a
no-op. Two things surprise people: lower-case `m` recalls memory while upper-case `M` stores it, so a
Shift state flips the effect (`key_classifier.rs:47`, `key_classifier.rs:48`); and there is no
delete-one-digit key, Backspace is a full clear the same as `AC` (`key_classifier.rs:41`).

### A click lands on the wrong button or nothing

Clicks are hit-tested against the 6x5 grid geometry in `hit_test` (`src/calc/layout.rs:38`). A click above
the grid, left of the padding, or in the gap band between two cells returns `None` and is idle, and the
gap rejection is deliberate so a click on the seam does not snap to a neighbour (`layout.rs:39`,
`layout.rs:52`). If clicks are off by a row or column, the padding, display height, and gap constants at
the top of `src/calc/layout.rs:19` are where the geometry is defined, and the renderer reads the same
`cell_size`/`cell_origin`, so a drift there would move the drawn buttons too (`layout.rs:26`).

### Display stuck on `Error`

This is the error latch, not a crash. Only `AC`, `MC`, and `MR` run while the latch is set; every other
operation returns early through the `is_error` guard (`src/calc/state.rs:53`, `src/calc/actions/clear.rs:20`,
`src/calc/actions/memory_recall.rs:19`). Press `AC` (or `c`, or Backspace) to reset, or `MR` to recall
memory and clear the latch in one step. The three causes are divide-by-zero, `sqrt` of a negative, and
`i128` overflow (`src/calc/op.rs:39`, `src/calc/unary.rs:34`, `src/calc/op.rs:32`). Because every operation
is checked, an error is always one of these three typed states, never a wrap or a trap.

### A result looks truncated or rounded

The formatter caps output at 32 bytes and at eight fractional digits, and trims trailing fractional zeros
when the user has not pinned a width, so a value with more precision than that is rounded in display, not
in the stored `i128` (`src/calc/format/display.rs:23`, `src/calc/format/fraction.rs:19`,
`src/calc/format/constants.rs:17`). The next operation still acts on the full-precision stored value, so a
displayed `0.33333333` continued in a calculation is more accurate than what the panel shows.

### Frame blank or stale with a responsive keypad

If clicks change the state (a digit lands, the badge appears) but the window shows nothing or a stale
frame, the split is between the state and the renderer. Every handled event returns
`EventOutcome::Repaint`, and `paint` reconstructs the whole surface from `State` each time
(`src/calc/event/on_key.rs:29`, `src/calc/paint/frame.rs:26`). A blank frame with a live state points at
the paint path or the skeleton's present step, not at the arithmetic.

## Source map

```
  src/userspace/init/capsule_boot/run.rs          [APP-CALCULATOR] capsule spawned / error path
  src/userspace/init/spawn_plan/apps.rs           the APP-CALCULATOR prefix and spawn entry
  src/userspace/capsule_calculator/spawn.rs       verified spawn, empty debug_tag, requested caps
  userland/capsule_calculator/src/calc/manifest.rs    the input subscription mask
  userland/capsule_calculator/src/calc/event/router.rs  key-down / button-down gate; Idle otherwise
  userland/capsule_calculator/src/calc/event/key_classifier.rs  the keymap and its ignore arm
  userland/capsule_calculator/src/calc/layout.rs  hit_test and the grid geometry constants
  userland/capsule_calculator/src/calc/state.rs   the is_error guard behind stuck-Error
  userland/capsule_calculator/src/calc/format/    the display formatting and its caps
  userland/capsule_calculator/src/calc/paint/     the frame renderer
```

Every reference above is verified against those trees.

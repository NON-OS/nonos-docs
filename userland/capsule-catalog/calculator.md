# capsule_calculator (full reference)

`capsule_calculator` is the calculator application in the NONOS userland tree: a small GUI window with a
five-column keypad and a fixed-point arithmetic engine behind it. It is deliberately the opposite of the
terminal. Where the terminal reaches the filesystem, the network, and the installer, the calculator
reaches nothing outside its own window. It is the cleanest least-privilege example in the app catalog:
the same window-and-input envelope as any GUI app, and no authority beyond it. The source is
`userland/capsule_calculator/`.

It is an [app-skeleton](../writing-an-app.md) GUI app. The kernel spawns it under service handle
`app.calculator` on service port 4720 with a reply port on 4721, and its capability mask is `0x1819`
(`userland/capsule_calculator/Capsule.mk:11`). Its sibling reference page is the exhaustive
[terminal reference](terminal/README.md); this page is the same standard for the calculator.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [User reference](#user-reference)
- [The evaluation model](#the-evaluation-model)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The calculator is an ordinary NONOS GUI application. Its entry point hands its `App` implementation to
the skeleton's `run`, so the runtime owns the surface, the window, the input subscription, and the paint
loop, and the calculator supplies three things: a manifest for a normal window, an `on_event` that turns
a keystroke or a pointer click into an arithmetic action, and a `paint` that draws the frame
(`src/main.rs:28`, `src/calc/app.rs:34`).

Behind the keypad is a fixed-point calculator. There is no expression parser and no operator precedence.
The engine is the classic single-pending-operator model of a physical desk calculator: a display value,
one pending operand, one pending operator, and a memory register, all held in one `State`
(`src/calc/state.rs:28`). Numbers are `i128` scaled by `10^8`, so the display carries eight fractional
digits, and every arithmetic step is a checked or saturating integer operation that turns overflow into a
typed error rather than a wrap or a panic (`src/calc/fixed.rs:17`, `src/calc/op.rs:29`). The calculator
is a signed capsule spawned as part of the desktop fleet at boot.

## Identity

Everything the kernel and the service registry need to name and reach the calculator comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `calculator` | `Capsule.mk:1` |
| Service handle | `app.calculator` | `Capsule.mk:2`, `src/userspace/capsule_calculator/spawn.rs:31` |
| Namespace | `systems.nonos.app.calculator` | `Capsule.mk:7` |
| Service endpoint | `service:4720:app.calculator` | `Capsule.mk:8`, `spawn.rs:32` |
| Reply endpoint | `reply:4721:endpoint.app.calculator.reply` | `Capsule.mk:9`, `spawn.rs:33`, `spawn.rs:34` |
| Capability mask | `0x1819` | `Capsule.mk:11` |
| Binary name | `calculator` | `Capsule.mk:5` |
| Kernel feature | `nonos-capsule-calculator` | `Capsule.mk:6` |
| Kernel mirror | `src/userspace/capsule_calculator` | `Capsule.mk:12` |

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

The kernel spawn path requests exactly those five capabilities by name and no others
(`src/userspace/capsule_calculator/spawn.rs:50`). There is no `Network` bit (4), no `FileSystem` bit
(64), no `Debug` bit (256), and no hardware, driver, MMIO, or DMA capability in the mask. The spawn spec
also sets an empty `debug_tag` (`spawn.rs:55`), so the capsule is granted no serial surface at all. This
is the whole basis of the security analysis below: the calculator can create a surface, ask the display
for its size, and speak IPC to the compositor through the skeleton, and nothing else.

## User reference

The calculator takes two kinds of input: pointer clicks on the on-screen keypad, and keyboard keys.
Both funnel into the same action set. A click is hit-tested to a grid cell and runs that button's action
(`src/calc/event/on_pointer_button.rs:24`); a key is classified to an action or ignored
(`src/calc/event/key_classifier.rs:26`). Every action runs through one dispatcher
(`src/calc/actions/dispatch.rs:24`), so a button and its keyboard shortcut are the exact same code path.

### The keypad

The keypad is a fixed 6-row by 5-column grid, assembled row by row (`src/calc/buttons/mod.rs:27`). Each
cell has a label, a visual role, and an action. This is the complete grid, top to bottom, left to right,
with the button label as it is painted:

| Row | Cells | Source |
|---|---|---|
| Memory | `MC`  `MR`  `M+`  `M-`  `MS` | `src/calc/buttons/row_memory.rs:19` |
| Function | `AC`  `+/-`  `%`  `sqrt`  `/` | `src/calc/buttons/row_function.rs:20` |
| Seven | `7`  `8`  `9`  `x^2`  `*` | `src/calc/buttons/row_seven.rs:20` |
| Four | `4`  `5`  `6`  `1/x`  `-` | `src/calc/buttons/row_four.rs:20` |
| One | `1`  `2`  `3`  `.`  `+` | `src/calc/buttons/row_one.rs:20` |
| Zero | `0`  `00`  `00`  `=`  `=` | `src/calc/buttons/row_zero.rs:19` |

Two cosmetic notes about the bottom row, both from source. The two `00` cells and the `0` cell all carry
`Action::Digit(0)`, so clicking `00` inserts a single `0`, not two (`row_zero.rs:20`). The two `=` cells
are both `Action::Equals`, so the equals key spans the bottom-right two columns as one wide button
(`row_zero.rs:23`).

### Keyboard input

Keys are classified in `key_classifier.rs`. Any code above `0x7F` is ignored, and Esc (`0x1B`) closes
the window; everything else maps to an action by its ASCII byte (`src/calc/event/key_classifier.rs:27`).
The full map:

| Key | Action | Source |
|---|---|---|
| `0`..`9` | insert that digit | `key_classifier.rs:34` |
| `.` | begin the fractional part | `key_classifier.rs:35` |
| `+` | set pending operator to add | `key_classifier.rs:36` |
| `-` | set pending operator to subtract | `key_classifier.rs:37` |
| `*`, `x`, `X` | set pending operator to multiply | `key_classifier.rs:38` |
| `/` | set pending operator to divide | `key_classifier.rs:39` |
| `=` or Enter (`0x0D`) | evaluate the pending operation | `key_classifier.rs:40` |
| `c`, `C`, or Backspace (`0x08`) | all-clear | `key_classifier.rs:41` |
| `n`, `N` | negate (sign flip) | `key_classifier.rs:42` |
| `%` | percent (divide display by 100) | `key_classifier.rs:43` |
| `r`, `R` | square root | `key_classifier.rs:44` |
| `q`, `Q` | square (x squared) | `key_classifier.rs:45` |
| `i`, `I` | reciprocal (1/x) | `key_classifier.rs:46` |
| `m` | memory recall | `key_classifier.rs:47` |
| `M` | memory store | `key_classifier.rs:48` |
| `a`, `A` | memory add (M+) | `key_classifier.rs:49` |
| `s`, `S` | memory subtract (M-) | `key_classifier.rs:50` |
| `l`, `L` | memory clear | `key_classifier.rs:51` |
| Esc (`0x1B`) | close the window | `key_classifier.rs:27` |
| anything else | ignored | `key_classifier.rs:52` |

Note that Backspace is not a single-digit delete; it maps to `Action::Clear`, the full all-clear, the
same as `AC` or `c` (`key_classifier.rs:41`, `src/calc/actions/clear.rs:20`). There is no delete-one-digit
operation in the current build.

### Operations

Every action below is dispatched from `dispatch::run` (`src/calc/actions/dispatch.rs:24`). Unless noted,
each one first checks the error latch and does nothing while the display shows `Error`, so a stuck error
can only be cleared by `AC` or memory recall.

Digit entry and the decimal point:

| Operation | Behavior | Source |
|---|---|---|
| Digit `0`..`9` | If a new number is starting, the digit replaces the display; otherwise it is appended, growing the integer part (capped at 16 integer digits) or the next fractional place (capped at 8) | `src/calc/actions/digit.rs:24` |
| `.` (decimal) | Switch entry into the fractional part; on a fresh number it starts from `0.` | `src/calc/actions/decimal.rs:19` |

Digit entry is where the fixed-point detail lives. On the first digit of a fresh number the display is
set to `digit * FRAC` and the new-input latch is cleared (`digit.rs:28`). After that, an integer digit is
folded in by pulling the value apart into integer and fractional parts, shifting the integer part up one
decimal place, and re-scaling, refusing once the integer part reaches 16 digits (`digit.rs:36`,
`INTEGER_DIGIT_LIMIT` at `digit.rs:22`). A fractional digit is placed at the next open fractional
position and refused once eight fractional digits are typed (`digit.rs:48`, `MAX_FRACTION_DIGITS` at
`src/calc/fixed.rs:20`). Every step uses `saturating_*` so a very long entry saturates instead of
wrapping.

Binary operators and evaluation:

| Operation | Behavior | Source |
|---|---|---|
| `+` `-` `*` `/` (set operator) | Commit any pending operation first (chaining), then latch the new pending operator and begin a fresh operand | `src/calc/actions/set_op.rs:20` |
| `=` / Enter (equals) | Apply the pending operator to the stored operand and the display, show the result, and clear the pending operator | `src/calc/actions/equals.rs:20` |

Setting an operator is where chained arithmetic comes from. If an operator is already pending and the
user has entered a new operand, `set_op` evaluates the pending step immediately and keeps the result as
the new operand, so `2 + 3 + 4` shows `5` after the second `+` and `9` after `=`
(`src/calc/actions/set_op.rs:24`). If that intermediate evaluation errors (for example an overflow), the
operator is dropped and the display latches `Error` (`set_op.rs:30`). Equals with no pending operator
does nothing (`equals.rs:21`).

Unary functions, each applied to the current display value:

| Operation | Behavior | Errors | Source |
|---|---|---|---|
| `+/-` (negate) | Flip the sign of the display | Overflow on `i128::MIN` | `src/calc/actions/negate.rs:20` |
| `%` (percent) | Divide the display by 100 | none | `src/calc/actions/percent.rs:20` |
| `x^2` (square) | Multiply the display by itself | Overflow | `src/calc/actions/square.rs:20`, `src/calc/unary.rs:20` |
| `sqrt` (square root) | Integer Newton square root of the display | DomainError on a negative input | `src/calc/actions/square_root.rs:20`, `src/calc/unary.rs:33` |
| `1/x` (reciprocal) | `FRAC^2 / display` | DivByZero on zero, Overflow | `src/calc/actions/reciprocal.rs:20`, `src/calc/unary.rs:25` |

Percent is a plain integer divide by 100 with no rounding, so `1 %` yields `0.01` and `5 %` yields
`0.05` (`percent.rs:23`). Square root of a negative value latches a domain error, and `sqrt(0)` is `0`
(`unary.rs:33`). Reciprocal of zero latches a divide-by-zero (`unary.rs:26`).

Memory register (the `M` row):

| Operation | Behavior | Errors | Source |
|---|---|---|---|
| `MS` (memory store) | Copy the display into memory | none | `src/calc/actions/memory_store.rs:19` |
| `MR` (memory recall) | Copy memory into the display, clearing any error latch and starting a new entry | none | `src/calc/actions/memory_recall.rs:19` |
| `M+` (memory add) | Add the display to memory | Overflow (memory is left unchanged) | `src/calc/actions/memory_add.rs:19` |
| `M-` (memory subtract) | Subtract the display from memory | Overflow (memory is left unchanged) | `src/calc/actions/memory_sub.rs:19` |
| `MC` (memory clear) | Zero the memory register | none | `src/calc/actions/memory_clear.rs:19` |

`MR` is the one operation that runs even while the display shows `Error`: it recalls memory, resets the
error latch to `None`, and starts a fresh input (`memory_recall.rs:19`). `MC` also runs unconditionally,
since it just zeroes the register (`memory_clear.rs:19`). `M+` and `M-` use `checked_add`/`checked_sub`
and refuse to corrupt the register on overflow, latching an error instead of writing a wrapped value
(`memory_add.rs:23`, `memory_sub.rs:23`). When the register is non-zero the paint layer draws a small
amber `M` badge in the top-left of the display (`src/calc/paint/memory_badge.rs:25`, `memory_engaged` at
`src/calc/state.rs:50`).

Clear:

| Operation | Behavior | Source |
|---|---|---|
| `AC` / `c` / Backspace | Zero the display and pending operand, drop the pending operator, and clear the error latch. Memory is not touched | `src/calc/actions/clear.rs:20` |

`AC` is the universal reset for the arithmetic state, but it deliberately leaves the memory register
alone; only `MC` clears memory (`clear.rs:20` versus `memory_clear.rs:19`).

### Error handling

Errors are a single latch on `State`, one of `None`, `DivByZero`, `DomainError`, or `Overflow`
(`src/calc/state.rs:20`). When any operation produces an error it sets the latch and zeroes the display
(for example `equals.rs:29`, `square.rs:26`). While the latch is set, the display paints the red text
`Error` instead of a number (`src/calc/paint/display.rs:34`, `ERROR_TEXT` at
`src/calc/format/constants.rs:18`), and every operation except `AC`, `MC`, and `MR` returns early without
acting (the `state.is_error()` guard at the top of each action, `state.rs:53`). The three ways to reach
the error state:

- Divide by zero: `n / 0 =` sets `DivByZero`, and so does `1/x` on zero (`src/calc/op.rs:39`,
  `src/calc/unary.rs:26`).
- Domain error: `sqrt` of a negative display value sets `DomainError` (`src/calc/unary.rs:34`).
- Overflow: any add, subtract, multiply, square, negate, or memory add/subtract that exceeds `i128`
  bounds sets `Overflow` through the `checked_*` path (`src/calc/op.rs:32`, `src/calc/unary.rs:21`,
  `src/calc/actions/negate.rs:26`).

The display can show at most 32 bytes; the formatter renders sign, integer part, a decimal point, and up
to eight fractional digits, trimming trailing zeros when the user has not pinned a fractional width
(`src/calc/format/display.rs:23`, `src/calc/format/fraction.rs:19`, `DISPLAY_MAX` at
`src/calc/format/constants.rs:17`).

## The evaluation model

There is no expression grammar. The whole engine is four fields updated in place
(`src/calc/state.rs:28`):

- `display` is the number currently shown and the number the next operation acts on.
- `operand` is the left-hand side saved when an operator is pressed.
- `operator` is the one pending operator, or `Op::None` (`src/calc/op.rs:20`).
- `memory` is the independent memory register.

A calculation is a sequence of updates: typing digits builds `display`, pressing an operator saves
`display` into `operand` and latches `operator` (evaluating any prior pending step first), and pressing
`=` computes `apply(operand, display, operator)` (`src/calc/op.rs:29`). `apply` is the only place the
four operators are defined: add and subtract are `checked_add`/`checked_sub`; multiply is a checked
multiply followed by a divide by `FRAC` to keep the fixed-point scale; divide scales the numerator up by
`FRAC` first and rejects a zero divisor (`op.rs:32`). Because there is one pending operator and no
precedence, input is evaluated strictly left to right, exactly like a four-function desk calculator.

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` calls the skeleton's `run(calc::Calculator::new)`
(`src/main.rs:27`). The single top-level module is `calc`, which re-exports `Calculator`
(`src/calc/mod.rs:31`). Under it the modules are `actions` (one file per operation), `buttons` (the grid),
`event` (the input router and classifiers), `format` (number to text), `paint` (the renderer), plus
`state`, `op`, `unary`, `fixed`, `layout`, `manifest`, and `theme` (`src/calc/mod.rs:17`).

The model is a `Calculator` that owns one `State` (`src/calc/app.rs:24`). `State` holds the display, the
pending operand, the pending operator, the memory register, the new-input latch, the count of fractional
digits typed, and the error latch (`src/calc/state.rs:28`). There is no heap-held document, no history,
and no shared static, so the entire machine is a handful of integers.

Lifecycle:

1. The kernel spawns the capsule at boot through the desktop fleet plan
   (`src/userspace/init/spawn_plan/apps.rs:21`, `apps.rs:59`), which verifies the embedded ELF, id cert,
   manifest, and attestation trailer, registers `app.calculator` on port 4720, and logs
   `[APP-CALCULATOR] capsule spawned` (`src/userspace/capsule_calculator/spawn.rs:40`,
   `src/userspace/init/spawn_plan/boot.rs:26`, `src/userspace/init/capsule_boot/run.rs:29`).
2. The skeleton `run` creates the window from the manifest (a 360x520 Normal window titled `Calculator`,
   subscribing to key-down, pointer button-down, and absolute pointer input) and drives the event and
   paint loop (`src/calc/manifest.rs:28`).
3. Each event flows in through `on_event`: a button-down goes to `on_pointer_button`, a key-down to
   `on_key`, and anything else is idle (`src/calc/event/router.rs:23`). A pointer click hit-tests to a
   grid cell and runs its action; a key classifies to an action or is ignored
   (`src/calc/event/on_pointer_button.rs:28`, `src/calc/event/on_key.rs:24`). Esc returns
   `EventOutcome::Close`; any handled action returns `EventOutcome::Repaint` (`on_key.rs:25`).
4. `paint` composes the frame in a fixed order: background, wordmark, the display value, the memory
   badge, then the keypad grid (`src/calc/paint/frame.rs:26`). The grid is drawn cell by cell with a
   per-role color (`src/calc/paint/grid.rs:23`, `src/calc/paint/button.rs:28`). The frame lands in the
   shared surface the compositor presents.

## Protocol and IPC

The calculator exposes no application opcodes of its own beyond what the app skeleton registers for it
(the `app.calculator` service on port 4720 and the reply inbox on 4721,
`src/userspace/capsule_calculator/spawn.rs:31`). It makes no outbound service calls of its own. Unlike
the terminal, there is no vfs client, no clipboard client, no network client, and no installer call
anywhere in the tree; a search of the capsule finds only the app-skeleton surface.

Everything it does that leaves the capsule is the app envelope the skeleton owns:

- Window registration and the per-frame paint buffer, requested from the toolkit endpoint through the
  skeleton's `run` (`src/main.rs:27`, manifest at `src/calc/manifest.rs:28`).
- Input delivery, which the skeleton hands to `on_event` as decoded `InputEvent`s
  (`src/calc/app.rs:38`, `userland/app_skeleton/src/input/event.rs:20`).
- Surface presentation, which the skeleton flushes to the compositor after a `Repaint` outcome.

The manifest declares the exact input subscription: key-down, pointer button-down, and absolute pointer
motion (`src/calc/manifest.rs:23`). No other IPC surface exists in the capsule.

## Security analysis

The calculator is the model least-privilege application in the catalog. Its authority is exactly the app
envelope and nothing more, and unlike the terminal it never even uses the parts of the envelope that
reach other services. Its mask `0x1819` grants CoreExec, IPC, Memory, GraphicsDisplayQuery, and
GraphicsSurfaceCreate (`Capsule.mk:11`, `src/userspace/capsule_calculator/spawn.rs:50`). There is no
Network bit, no FileSystem bit, no Debug bit, and no hardware, driver, MMIO, or DMA capability. The
calculator cannot read a block device, open a socket, resolve a name, touch a device register, or write
a log line.

That last point is deliberate. The spawn spec sets an empty `debug_tag` (`spawn.rs:55`) and the mask
omits `Debug` (256), so the kernel grants the capsule no serial surface and the capsule emits no debug
markers of its own. It leaves no trace on the wire beyond the frames the compositor already presents.

The privacy posture follows from the code, not just the manifest. The entire machine is the `State`
struct of a few integers (`src/calc/state.rs:28`); there is no file it reads, no file it writes, no
socket, no persistent identifier, and no history buffer. Every operand exists only in process memory and
vanishes the moment the user presses `AC` or the window closes. A compromise of the calculator gains an
attacker exactly the five capabilities above: the right to draw in its own window and ask the display its
size. It cannot pivot to the filesystem, the network, or another capsule, because it never held the
authority to ask.

The arithmetic itself is also hardened against undefined behavior. There is no `unsafe` outside the
`_start` extern, and every operation uses `checked_*` or `saturating_*` so that overflow, division by
zero, and a negative square root become a typed `ErrorKind` and a red `Error` display rather than a wrap,
a trap, or a panic (`src/calc/op.rs:29`, `src/calc/unary.rs:20`, `src/calc/state.rs:20`). The release
profile is `panic = "abort"` (`Cargo.toml:26`), so even an unexpected panic terminates the capsule rather
than unwinding; the point of the checked arithmetic is that no reachable input reaches that path.

## How to contribute

The source lives at `userland/capsule_calculator/`. The arithmetic is under `src/calc/actions/`,
`src/calc/op.rs`, and `src/calc/unary.rs`; the keypad is under `src/calc/buttons/`; the input path is
under `src/calc/event/`; the renderer is under `src/calc/paint/`; and the fixed-point and formatting
primitives are `src/calc/fixed.rs` and `src/calc/format/`.

To add a new operation:

1. Write the action module. Each operation is one file under `src/calc/actions/`, exposing a
   `pub fn run(state: &mut State)` (or `run(state: &mut State, ...)` for digit and operator) that mutates
   `State`. Guard the top with `if state.is_error() { return; }` unless the operation should run while
   errored, the way `memory_recall` and `memory_clear` do (`src/calc/actions/memory_recall.rs:19`). Do
   the math with `checked_*`/`saturating_*` and set `state.error` on failure rather than panicking, the
   way `square` does (`src/calc/actions/square.rs:24`). If it is a pure numeric transform, put the math in
   `src/calc/unary.rs` and keep the action file thin.
2. Register it. Add a variant to the `Action` enum (`src/calc/buttons/kinds.rs:28`) and a match arm in the
   dispatcher (`src/calc/actions/dispatch.rs:24`), and declare the module in
   `src/calc/actions/mod.rs:17`.
3. Give it a way in. Add a keyboard mapping in the classifier (`src/calc/event/key_classifier.rs:33`) and
   place the button on the keypad by editing the relevant row file under `src/calc/buttons/` (for example
   `src/calc/buttons/row_function.rs:20`); the grid is a fixed 6x5, so a new button replaces an existing
   cell unless the grid dimensions change (`src/calc/buttons/mod.rs:27`, `src/calc/layout.rs:21`).

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_calculator/Capsule.mk:14`):

```
  make nonos-mk-calculator                build the capsule ELF
  make nonos-mk-calculator-sign           produce the id cert, manifest, and attestation trailer
  make nonos-mk-calculator-verify         verify the signed artifacts against the trust anchor
  make nonos-mk-check-calculator-keys     check the per-capsule signing keys exist
```

For a running desktop that includes the calculator, `make nonos-mk-calculator-prod` builds the full
desktop GUI image (`Makefile:1163`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`,
`expect`, `todo!`, or `unimplemented!` in capsule code (every failure becomes a typed `ErrorKind` and a
`false`-free early return, never a panic; the release profile is `panic = "abort"`, `Cargo.toml:26`);
modular files, one unit per file, with `mod.rs` used only for re-exports (`src/calc/mod.rs`,
`src/calc/actions/mod.rs`); no `unsafe` outside the unavoidable `_start` extern (`src/main.rs:27`); and
the AGPL header at the top of every source file, matching the header on every existing module.

## Debugging

The calculator holds no `Debug` capability and emits no serial markers of its own, so the only log
evidence is the kernel-side boot line. On a successful boot the kernel prints `[APP-CALCULATOR] capsule
spawned` (tag `APP-CALCULATOR`, message `capsule spawned`) from the boot log
(`src/userspace/init/spawn_plan/apps.rs:62`, `src/userspace/init/capsule_boot/run.rs:29`). An absent line
means the capsule never started, usually a signature, manifest, or capability failure; the error path
prints an `[ERROR]` line built from the spawn error instead (`src/userspace/init/capsule_boot/run.rs:32`).

Failure modes and where to look:

- Window opens but nothing responds. The window subscribes to key-down, pointer button-down, and absolute
  pointer input (`src/calc/manifest.rs:23`); `on_event` drops any other event kind as idle
  (`src/calc/event/router.rs:27`). If keys and clicks both do nothing, the input path into the app
  (compositor, wm, input_router) is the suspect, not the calculator.
- A key does nothing. The classifier ignores every code above `0x7F` and every ASCII byte not in its map
  (`src/calc/event/key_classifier.rs:30`, `key_classifier.rs:52`), so an unmapped key is expected to be a
  no-op. There is no delete-one-digit key; Backspace is a full clear (`key_classifier.rs:41`).
- A click lands on the wrong button or nothing. Clicks are hit-tested against the 6x5 grid geometry in
  `hit_test` (`src/calc/layout.rs:38`); a click in the padding or the display area returns `None` and is
  idle (`src/calc/event/on_pointer_button.rs:29`). If a click is off by a row, the padding and gap
  constants in `src/calc/layout.rs:19` are where the geometry is defined.
- Display stuck on `Error`. This is the error latch, not a crash. Only `AC`, `MC`, and `MR` run while the
  latch is set; every other operation returns early (`src/calc/state.rs:53`, `src/calc/actions/clear.rs:20`,
  `src/calc/actions/memory_recall.rs:19`). Press `AC` (or `c`, or Backspace) to reset. The three causes
  are divide-by-zero, sqrt of a negative, and `i128` overflow (`src/calc/op.rs:39`,
  `src/calc/unary.rs:34`, `src/calc/op.rs:32`).
- A result looks truncated. The formatter caps output at 32 bytes and at eight fractional digits and
  trims trailing fractional zeros, so a value with more precision than that is rounded in display, not in
  the stored `i128` (`src/calc/format/display.rs:23`, `src/calc/format/fraction.rs:19`).

## Source map

```
  src/main.rs                              _start -> run(Calculator::new)
  src/calc/mod.rs                          the module tree; re-exports Calculator
  src/calc/app.rs                          Calculator: owns one State; App impl (manifest/on_event/paint)
  src/calc/state.rs                        State: display, operand, operator, memory, error latch
  src/calc/fixed.rs                        i128 fixed-point scale (FRAC = 1e8, 8 fractional digits)
  src/calc/op.rs                           Op enum and apply(): the four binary operators
  src/calc/unary.rs                        square, reciprocal, integer sqrt
  src/calc/actions/                        one file per operation (digit, set_op, equals, memory_*, ...)
  src/calc/actions/dispatch.rs             Action -> handler dispatch
  src/calc/buttons/                        the 6x5 keypad grid, one file per row + kinds
  src/calc/event/                          router, key_classifier, on_key, on_pointer_button
  src/calc/format/                         number-to-text (integer, fraction, display, constants)
  src/calc/paint/                          the frame renderer (background, wordmark, display, badge, grid, button)
  src/calc/layout.rs                       grid/display geometry and hit_test
  src/calc/manifest.rs                     window title, size, and input subscription mask
  src/calc/theme.rs                        the phosphor-green palette
  Capsule.mk                               slug, handle, ports, capability mask, kernel mirror
  src/userspace/capsule_calculator/        the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/apps.rs    the desktop-fleet spawn entry
  nonos-mk/capsule.mk                      the generated nonos-mk-calculator[-sign|-verify] targets
```

Every reference above is verified against those trees.

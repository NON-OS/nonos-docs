# Cursor and input subscription

This page mirrors the cursor and focus paths that cross `src/server/handlers/`, `src/state/`, and
`src/frame_pacer/cursor.rs`: how the pointer is moved and drawn, and how a capsule marks itself focused. It
is the smallest pillar, and it is the one with the most honest gaps in v1. The wire ops are in
[operations.md](operations.md); the cursor tracker is in [scene-and-damage.md](scene-and-damage.md); the
sprite blit is in [frame-pacing.md](frame-pacing.md). Back to the [README](README.md).

## Moving the cursor

The [input router](../input-router/README.md) turns pointer motion into `CURSOR_UPDATE` calls; the
compositor owns where the cursor is drawn. The handler (`src/server/handlers/cursor_update.rs:22`) takes a
16-byte payload of `x, y, visible` as three `u32`, `visible` non-zero meaning shown (`cursor_update.rs:32`
through `:41`). A wrong length or an off-screen position is dropped silently, returning `Ok` with no reply,
so a bad update never stalls the drain and never damages anything (`cursor_update.rs:29`, `:42`).

On a valid update it calls `ctx.cursor.update`, which returns the previous cursor state
([scene-and-damage.md](scene-and-damage.md)). It then damages the previous cursor cell if it was visible,
and the new cell if the cursor is now visible, each a `CURSOR_SIDE = 32` pixel box clipped to the screen
edges with a saturating subtract (`cursor_update.rs:20`, `:45` through `:55`). Damaging both cells is what
erases the old cursor and paints the new one on the next frame: the old cell repaints to background plus
whatever layer is under it, and the new cell gets the sprite.

Note that `CURSOR_UPDATE` does not send a status reply. It is the one op whose handler returns `Ok` without
calling `respond`, because the input router fires it at pointer rate and does not wait on a reply
(`cursor_update.rs:30`).

## Drawing the cursor

The sprite itself is drawn in the compositing pass, not the handler. If `ctx.cursor.current().visible` is
set, `paint` calls the cursor blit with the current position, clipped to the frame's damage rectangle
(`src/frame_pacer/composite.rs:61`). The sprite is a 14-pixel white arrow with a one-pixel black shadow, two
pixels thick, drawn with clipped volatile writes (`src/frame_pacer/cursor.rs:24`,
[frame-pacing.md](frame-pacing.md)). It is a software blit, not a hardware sprite, so the cursor moves only
as fast as the frame it is composited into.

The cursor starts at the screen centre: setup constructs the tracker with
`CursorTracker::at(width / 2, height / 2)` on both backends (`src/setup/prime_once.rs:70`,
`src/setup/prime_gop.rs:83`).

## Focus and input subscription

Two ops write the focus table, and both are recorded rather than acted on in v1.

- `INPUT_SUBSCRIBE` (`src/server/handlers/input_subscribe.rs:21`) takes no payload and marks the sender
  itself focused: `ctx.focus.set(sender_pid)` (`input_subscribe.rs:27`). It is a capsule saying "I want
  input." v1 does not fan input out through the compositor; the input router does the actual delivery, so
  this call records intent without yet routing keys.
- `FOCUS_SET` (`src/server/handlers/focus_set.rs:21`) takes an 8-byte payload whose first `u32` is a target
  pid and stores it: `ctx.focus.set(target_pid)` (`focus_set.rs:34`). This is a capsule (the wm) telling the
  compositor which pid should hold focus.

The `FocusTable` keeps a single focused pid (`src/state/focus.rs:22`). Its stated role is to record who owns
input dispatch so the input router can be queried without a round trip through the wm (`focus.rs:17`). The
recorded pid is not yet used to highlight a focused window or to gate anything the compositor draws; it is a
value other capsules can read back.

## Honest gaps

Stated from the code:

- `INPUT_SUBSCRIBE` records the caller as focused but v1 does not fan input out through the compositor
  (`src/server/handlers/input_subscribe.rs:27`).
- The pid set by `FOCUS_SET` is recorded but not yet used to highlight a focused window
  (`src/state/focus.rs:31`).
- The cursor is a software blit, not a hardware sprite (`src/frame_pacer/cursor.rs`).
</content>

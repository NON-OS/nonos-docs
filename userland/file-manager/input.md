# Input

This page mirrors the event units under `src/fm/`: the router (`event_dispatch.rs`, `event_mode.rs`),
the browse keys (`event_browse.rs`), the mode switches (`event_modes.rs`), the mouse row select
(`event_mouse.rs`), and the cursor and scroll movement (`event_move.rs`, `event_open.rs`,
`event_parent.rs`, `scroll.rs`). It is the exhaustive keybinding and pointer reference. The actions the
browse keys reach and the prompts they open are on [actions.md](actions.md); the modes they switch into
are on [listing.md](listing.md) (filter) and this page (preview and help live in their own units).

## The router

Input arrives as key-down events and absolute pointer button-down events. `on_event` runs three steps in
order (`src/fm/event_dispatch.rs:24`):

1. `select_row` moves the cursor to a clicked row while browsing (`event_dispatch.rs:25`).
2. Anything that is neither a button-down nor a key-down is dropped as `Idle` (`event_dispatch.rs:26`).
3. `route` hands the event to the mode-specific handler if the app is in a non-browse mode; otherwise
   the event falls through to the browse keys (`event_dispatch.rs:29`, `:33`).

A pointer button-down while browsing is folded into the same path as Enter: `select_row` picks the row
under the pointer, then the click is dispatched as `KEY_ENTER`, so a single click opens the entry it
lands on (`event_dispatch.rs:32`, `event_mouse.rs:22`).

`route` dispatches by the current mode: Filter to `filter::on_key`, Help to `help::on_key`, Prompt to
`prompt::on_key`, Preview to `preview_key::on_key`, and Browse falls through by returning `None`
(`src/fm/event_mode.rs:26`).

## The mouse row select

`select_row` only acts on a button-down while browsing (`src/fm/event_mouse.rs:23`). It converts the
pointer y into a row against the shared geometry: `rel = y - FIRST_ROW_Y + 4`, and the row is
`scroll + rel / ROW_HEIGHT`, clamped to the entry count (`event_mouse.rs:26`, `:30`). A click above the
first row is ignored (`event_mouse.rs:27`). The `+ 4` and the `FIRST_ROW_Y`/`ROW_HEIGHT` constants are
the same ones the renderer draws with, so a click lands on the row it visually points at
(`src/fm/layout.rs:19`).

## Browse mode

While browsing, `on_browse_key` first offers the code to `run_action` (the file actions) and then to
`open_mode` (the mode switches), and only if neither claims it does it fall through to the navigation
keys (`src/fm/event_browse.rs:30`).

Navigation and opening (`src/fm/event_browse.rs:33`):

| Key | Action | Handler |
|-----|--------|---------|
| Up / `k` | move the cursor up one row | `event_browse.rs:37`, `:40`, `event_move.rs:22` |
| Down / `j` | move the cursor down one row | `event_browse.rs:38`, `:39`, `event_move.rs:22` |
| Enter / Right / `l` | open: enter a directory, or preview a file | `event_browse.rs:35`, `:42`, `event_open.rs:23` |
| Backspace / Left / `h` | go up to the parent directory | `event_browse.rs:36`, `:41`, `event_parent.rs:22` |
| left mouse click on a row | select that row and open it | `event_dispatch.rs:32`, `event_mouse.rs:22` |
| Esc | close the window | `event_browse.rs:34` |

The file actions (space, `a`, `c`, `x`, `p`, `o`, `u`, `s`) and the prompt keys (`n`, `m`, `r`, `d`)
are claimed before these navigation keys are reached; they are documented on [actions.md](actions.md).
The mode switches `/` and `?` are handled by `open_mode` (`src/fm/event_modes.rs:21`): `/` enters
incremental filter mode (`event_modes.rs:23`) and `?` opens the full-window help
(`event_modes.rs:27`).

## Opening, going up, and cursor movement

Opening a directory replaces the prefix, resets the cursor and scroll to the top, and refreshes the
listing; opening a file hands off to `open_preview` (`src/fm/event_open.rs:24`). Going up trims the
prefix to its parent and refreshes, and at `/` it does nothing (`src/fm/event_parent.rs:23`, `:26`).

The cursor move clamps down with `saturating_sub` and up against the entry count with a `min`, then
calls `ensure_visible` to keep the scroll window positioned (`src/fm/event_move.rs:24`, `:27`, `:29`).
`ensure_visible` slides the window so the cursor stays on screen: it pages up when the cursor moves
above the top row and pages down when it falls past the last of `LIST_VISIBLE = 7` rows
(`src/fm/scroll.rs:23`, `src/fm/layout.rs:21`).

## Preview and help keys

Preview mode is entered by opening a file; in it Up and Down scroll the visible window and Esc returns
to browsing (`src/fm/preview_key.rs:24`). The rest of the preview lives on [preview.md](preview.md).
Help mode is entered by `?`; any key dismisses it back to browsing (`src/fm/help.rs:27`). The help text
is drawn by the renderer and covered on [rendering.md](rendering.md).

## Source map

Everything here is drawn from the event units under `userland/capsule_file_manager/src/fm/`
(`event_dispatch.rs`, `event_mode.rs`, `event_modes.rs`, `event_browse.rs`, `event_mouse.rs`,
`event_move.rs`, `event_open.rs`, `event_parent.rs`, `scroll.rs`), the shared geometry in `layout.rs`,
and the mode handlers they route to (`preview_key.rs`, `help.rs`). Every reference above is verified
against those trees.

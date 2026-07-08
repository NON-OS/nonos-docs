# capsule_clipboard

The clipboard holds copied content with a bounded history and a copy-and-paste protocol, and it clears
itself after a period of inactivity. Service `clipboard` on port 4414, capability mask `0x19`. The source
is `userland/capsule_clipboard/`.

## The server loop

`main.rs:13` initializes the heap and runs the loop (`src/server/runner.rs:27`), which garbage-collects
on idle at the top of each iteration:

```
  run():
      clipboard = Clipboard::new(depth 16, total 256 KiB, idle timeout 600 s)
      loop:
          clipboard.expire_if_idle(now)          // clear the whole history if idle too long
          mk_ipc_recv_from(port 4414, &sender_pid)
          route(clipboard, payload, now)
          mk_ipc_reply(sender_pid, out)
```

The frame is `CBC0` (magic `0x43424930`), version 1.

## The operations

Seven operations (`src/protocol/ops.rs:17`):

```
  HEALTHCHECK=1  COPY=2  PASTE=3  HISTORY_LIST=4  HISTORY_GET=5  CLEAR=6  SET_IDLE_TIMEOUT=7
```

`COPY` (`src/server/handlers/copy.rs:21`) pushes a content-typed entry to the front of a FIFO, bounded at
16 entries and 256 KiB total (evicting the oldest when either bound is exceeded) and rejecting an entry
over 64 KiB. `PASTE` returns the most recent entry of a content type and touches the activity timestamp.
`SET_IDLE_TIMEOUT` adjusts the idle window within 5 seconds to 24 hours.

## The idle wipe

The defining behavior is the idle garbage collection (`src/state/clipboard/timer.rs:26`): if the
clipboard has been untouched for the idle timeout (600 seconds by default), the entire history is
cleared. Both copy and paste touch the activity timestamp, so the clipboard does not expire while it is
in use, but a clipboard left idle empties itself. This is a privacy posture: copied content does not
linger indefinitely.

## Honest scope

The `Clipboard` (`src/state/clipboard/types.rs:21`) is a `VecDeque` of entries with a running byte total
and the activity timestamp. Honest limits: the idle timeout clears the whole history at once rather than
per entry, the entries are stored in plaintext in RAM, and there is no per-caller isolation, the
clipboard is system-wide, so any capsule that can reach the service can read the history.

## Source

```
  userland/capsule_clipboard/src/server/runner.rs        the loop + idle GC
  userland/capsule_clipboard/src/server/handlers/         copy, paste, history, clear
  userland/capsule_clipboard/src/state/clipboard/         the FIFO, the idle timer
```

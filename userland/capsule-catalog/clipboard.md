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

## Security analysis

The mask is `0x19` (`CAPSULE_REQUIRED_CAPS` in `userland/capsule_clipboard/Capsule.mk`), which decodes to `CoreExec | IPC | Memory` against `src/capabilities/types.rs`. That is the minimum a service leaf needs: run, receive and reply over IPC, and hold a heap. It has no graphics, no filesystem, no network, no hardware, and no crypto. The clipboard never touches the screen, never persists to disk, and never reaches a device, so the whole capsule is a byte store behind an IPC port.

- **No persistence.** The mask carries no `FileSystem`, and the store is a `VecDeque` in RAM (`src/state/clipboard/types.rs:21`). Copied content lives only in the capsule's heap and is gone when it exits, and the idle wipe (`src/state/clipboard/timer.rs:26`) empties the whole history after the timeout even while the capsule keeps running.
- **Bounded, so it cannot be flooded.** `COPY` (`src/server/handlers/copy.rs:21`) evicts the oldest entry when either the 16-entry depth or the 256 KiB total is exceeded and rejects any single entry over 64 KiB, so a caller cannot grow the store without bound or exhaust the heap with one paste.
- **Honest boundary: no per-caller isolation.** The store is system-wide and the handlers do not check `sender_pid` against the entry owner, so any capsule that can reach port 4414 can `PASTE` or `HISTORY_LIST` whatever another capsule copied. The clipboard's confidentiality is the idle wipe and the fact that reaching the port at all requires the `IPC` capability, not per-entry ownership.

## Debugging

The service binds directly to port 4414 with `mk_ipc_recv_from` (`src/server/runner.rs:24`, `SERVICE_PORT`), rather than registering a name and then receiving, so a client reaches it by looking up `clipboard` in the service registry (`mk_service_lookup`) and sending to the port that resolves. The kernel prints the spawn marker as the capsule comes up:

```
  [SPAWN] name=clipboard pid=0x... caps=0x19 entry=0x...
```

`caps=0x19` in that line confirms the capsule was admitted with exactly `CoreExec | IPC | Memory` and nothing was added at load. The clipboard is also one of the names the spawn tracer prints extra install-stage lines for (`src/kernel_core/process_spawn/capsule_spawn/runner/install/trace.rs:20`), so a stall during its install shows up as a `[SPAWN] clipboard ...` stage line rather than silence.

Once up, the failure signatures are on the wire, not the console: a `COPY` of an oversized entry returns the error status rather than storing it, an op outside 1..7 is rejected by `route`, and a caller that finds the history empty after leaving the machine idle is seeing the idle wipe, not a crash. If a client's `mk_service_lookup("clipboard")` returns a zero port or pid the service never registered (the capsule failed to spawn), which the client surfaces as its own lookup error rather than an error from the clipboard.

## Source map

```
  userland/capsule_clipboard/src/server/runner.rs         the port-4414 loop and idle GC
  userland/capsule_clipboard/src/server/handlers/copy.rs  bounds check + FIFO push
  userland/capsule_clipboard/src/server/handlers/         paste, history, clear, set_idle_timeout
  userland/capsule_clipboard/src/state/clipboard/types.rs the VecDeque, byte total, activity timestamp
  userland/capsule_clipboard/src/state/clipboard/timer.rs the idle-wipe timer
  userland/capsule_clipboard/Capsule.mk                   CAPSULE_REQUIRED_CAPS = 0x19, endpoint 4414
```

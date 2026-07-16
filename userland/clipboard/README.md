# capsule_clipboard

The clipboard holds copied content with a bounded history and a copy-and-paste protocol, and it clears
itself after a period of inactivity. Service `clipboard` on port 4414, capability mask `0x19`. The source
is `userland/capsule_clipboard/`.

It is a userland service that runs at CPL 3. Any capsule that needs cut, copy, or paste talks to it over
IPC; the kernel never carries clipboard bytes itself. The copying capsule sends `OP_COPY` with a content
type and a byte string, and the pasting capsule sends `OP_PASTE` (or a history op) to recall it. Nothing
about the store touches the screen, the disk, or a device.

## The server loop

`main.rs:13` is the entry point. It initializes the userland heap through `nonos_libc::heap_init` and
exits with status `1` if that fails (`src/main.rs:14`), otherwise it hands control to `server::run`, which
never returns. The loop lives in `src/server/runner.rs:27`:

```
  run():
      clipboard = Clipboard::new(MAX_DEPTH 16, MAX_TOTAL_BYTES 256 KiB, DEFAULT_IDLE_TIMEOUT_MS 600000, now)
      in_buf  = vec![0; IPC_PAYLOAD_MAX]      // 64 KiB + 32
      out_buf = vec![0; IPC_PAYLOAD_MAX]
      loop:
          clipboard.expire_if_idle(read_time())          // clear the whole history if idle too long
          received = mk_ipc_recv_from(port 4414, in_buf, timeout 0, &sender_pid)
          if received <= 0 or sender_pid == 0: mk_yield(); continue
          n = route(&mut clipboard, &in_buf[..received], &mut out_buf, read_time())
          if n > 0: mk_ipc_reply(sender_pid, &out_buf[..n])
```

The receive timeout is `RECV_TIMEOUT_MS = 0` (`src/server/runner.rs:25`), so a receive that returns
nothing or a zero sender pid yields the CPU and loops rather than spinning. `read_time` (`src/server/runner.rs:55`)
reads `mk_time_millis` and floors a negative return to `0`, so a clock error cannot make the idle math run
backwards. The idle check runs at the top of every iteration, before the blocking receive, so a clipboard
that sits untouched past its timeout is emptied on the next wakeup.

The kernel side of the wire is thin. `spawn_clipboard_capsule` (`src/userspace/capsule_clipboard/spawn.rs:37`)
registers the service name `clipboard` on port 4414 and a reply inbox `endpoint.clipboard.reply` on port
4415, then verifies and launches the embedded, signed ELF. The endpoint registration
(`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:50`) stamps `Capability::IPC` on
the endpoint, so a caller needs the IPC capability even to send to the port.

## The wire format

Every request and reply starts with a fixed 20-byte header (`HDR_LEN = 20`, `src/protocol/header.rs:19`),
all fields little-endian:

```
  offset  size  field
  0       4     magic        0x43424930
  4       2     version      1
  6       2     op
  8       2     flags
  10      2     reserved     (zeroed in replies)
  12      4     request_id
  16      4     payload_len
```

The magic is `MAGIC = 0x4342_4930` (`src/protocol/header.rs:17`), which is the ASCII bytes `43 42 49 30`,
that is `CBI0`, the NONOS clipboard interface tag. `VERSION = 1` (`src/protocol/header.rs:18`). The client
side uses the same constant under the name `NCLP_MAGIC` (`userland/app_skeleton/src/wire/constants.rs:21`).

`parse` (`src/protocol/decode.rs:19`) decodes a request by explicit little-endian indexing, never by
`try_into().unwrap()`. It checks in order: the buffer holds at least the 20-byte header
(`E_BAD_LEN`), the magic matches (`E_BAD_MAGIC`), the version matches (`E_BAD_VERSION`), and the buffer
holds `HDR_LEN + payload_len` bytes (`E_BAD_LEN`). On success it returns the parsed `Request { op, flags,
request_id }` and a slice over exactly the declared payload. The op and request id are read before the
magic and version checks, so an error reply still echoes the caller's op and request id.

Every reply is built by `respond` (`src/server/respond.rs`). `respond::status` writes the header plus a
4-byte `status: i32` and returns `HDR_LEN + STATUS_LEN` = 24 bytes; `respond::with_payload` writes the
header, the status, and `payload_extra` more bytes. The reply header
(`src/protocol/encode.rs:19`) copies the request's op, flags, and request id back, zeroes the reserved
field, and sets `payload_len` to `STATUS_LEN + payload_extra`. The status is `0` on success or a negative
errno on failure (`src/protocol/encode.rs:29`). So a client can always read a signed `i32` status at
offset 20 and, for the read ops, op-specific data after it.

## The operations

Seven operations, dispatched by `route` (`src/server/handlers/router.rs:25`). The opcodes are in
`src/protocol/ops.rs:17`:

```
  OP_HEALTHCHECK    = 0x0001
  OP_COPY           = 0x0002
  OP_PASTE          = 0x0003
  OP_HISTORY_LIST   = 0x0004
  OP_HISTORY_GET    = 0x0005
  OP_CLEAR          = 0x0006
  OP_SET_IDLE_TIMEOUT = 0x0007
```

Any op outside this set returns `E_BAD_OP` (`src/server/handlers/router.rs:38`) with no state change.

`OP_HEALTHCHECK` (`src/server/handlers/health.rs:20`) is a liveness ping. It touches no state, does not
touch the activity timestamp, and returns status `0` and nothing else.

`OP_COPY` (`src/server/handlers/copy.rs:21`) takes a payload of `content_type: u32` followed by the
content bytes. It rejects a payload shorter than 4 bytes with `E_INVAL`, and rejects content longer than
`MAX_ENTRY_BYTES = 64 KiB` with `E_RANGE` (`src/server/handlers/copy.rs:27`). Otherwise it pushes the
entry to the front of the FIFO and returns status `0`. The push (`src/state/clipboard/storage.rs:21`)
adds the entry's bytes to the running total, then evicts from the back while either the 16-entry depth or
the 256 KiB total is exceeded, and updates the activity timestamp.

`OP_PASTE` (`src/server/handlers/paste.rs:21`) takes a `content_type: u32`, touches the activity
timestamp, and returns the most recent entry of that content type. The reply payload after the status is
`len: u32` followed by the data (`src/server/handlers/paste.rs:36`). If no entry of that type exists, it
returns status `0` with no data (`src/server/handlers/paste.rs:29`), so an empty clipboard is a success,
not an error. If the entry does not fit in the output buffer it returns `E_RANGE` and keeps the entry.
A short payload (under 4 bytes) is `E_INVAL`.

`OP_HISTORY_LIST` (`src/server/handlers/history_list.rs:21`) takes no payload and does not touch the
timestamp. It returns `count: u32` followed by `count` pairs of `(content_type: u32, len: u32)`, one per
stored entry in front-to-back order. If the list would not fit in the output buffer it returns `E_RANGE`.
This op exposes the shape of the whole history without exposing the bytes; the bytes come from
`OP_HISTORY_GET`.

`OP_HISTORY_GET` (`src/server/handlers/history_get.rs:21`) takes an `index: u32`, touches the activity
timestamp, and returns the entry at that index as `content_type: u32`, `len: u32`, then the data
(`src/server/handlers/history_get.rs:36`). Index 0 is the most recent entry. An out-of-range index is
`E_RANGE` (`src/server/handlers/history_get.rs:29`), a payload under 4 bytes is `E_INVAL`, and an entry
that does not fit the output buffer is `E_RANGE`.

`OP_CLEAR` (`src/server/handlers/clear.rs:21`) drops every entry and resets the byte total, then touches
the activity timestamp, and returns status `0`. There is no payload.

`OP_SET_IDLE_TIMEOUT` (`src/server/handlers/set_idle_timeout.rs:23`) takes a `timeout_ms: u64`. A payload
under 8 bytes is `E_INVAL`. The value `0` is accepted and disables the idle wipe. Any nonzero value must
fall within `MIN_IDLE_TIMEOUT_MS = 5000` (5 seconds) and `MAX_IDLE_TIMEOUT_MS = 24 * 60 * 60 * 1000` (24
hours) or it returns `E_RANGE` (`src/server/handlers/set_idle_timeout.rs:31`) and leaves the current
timeout unchanged. On success it updates the timeout and returns status `0`. Note this op does not touch
the activity timestamp.

## The data model

Each entry (`src/state/entry.rs:19`) is a `content_type: u32` and a `Vec<u8>` of the bytes. The content
type is an opaque tag; the capsule never interprets it and stores the bytes unchanged. The one type in use
today is `CONTENT_TYPE_TEXT = 1`, set by the client helpers (`userland/app_skeleton/src/clients/clipboard/copy.rs:23`,
`.../paste.rs:23`). Any other u32 works the same way; `OP_PASTE` matches on the exact type value, so two
callers using different type tags keep separate most-recent entries in the same FIFO.

The store (`src/state/clipboard/types.rs:21`) is a `VecDeque<Entry>` with a running `total_bytes`, the
depth and byte caps, the last-activity timestamp, and the idle timeout, all in the capsule's heap. The
bounds come from `src/protocol/limits.rs`:

```
  MAX_DEPTH             = 16
  MAX_TOTAL_BYTES       = 256 * 1024        (256 KiB)
  MAX_ENTRY_BYTES       = 64 * 1024         (64 KiB, per-entry cap)
  IPC_PAYLOAD_MAX       = MAX_ENTRY_BYTES + 32   (recv/reply buffer size)
  DEFAULT_IDLE_TIMEOUT_MS = 600000          (10 minutes)
  MIN_IDLE_TIMEOUT_MS   = 5000
  MAX_IDLE_TIMEOUT_MS   = 86400000          (24 hours)
```

The FIFO evicts from the back on `OP_COPY` when either the depth or the byte total is exceeded, using a
saturating subtract when it decrements the total, so the counter never underflows even under an
unexpected sequence.

## The idle wipe

The defining behavior is the idle garbage collection (`src/state/clipboard/timer.rs:26`). `expire_if_idle`
returns early without clearing if the timeout is `0` or the store is already empty. Otherwise, if the
elapsed time since the last activity (`idle_for`, `src/state/clipboard/timer.rs:23`, a saturating
subtract) is at least the idle timeout, it clears the whole history. `OP_COPY`, `OP_PASTE`, and
`OP_HISTORY_GET` all touch the activity timestamp, and `OP_CLEAR` touches it after wiping, so the
clipboard does not expire while it is in use. `OP_HEALTHCHECK`, `OP_HISTORY_LIST`, and
`OP_SET_IDLE_TIMEOUT` deliberately do not touch it, so listing the history or pinging liveness does not
extend the retention window. A clipboard left idle for the timeout empties itself. This is a privacy
posture: copied content does not linger indefinitely.

## How GUI capsules use it

Clients do not build the wire by hand. They go through `nonos_app_skeleton`, which exposes
`clipboard_copy` and `clipboard_paste` (`userland/app_skeleton/src/clients/clipboard/mod.rs`).

`clipboard_copy` (`userland/app_skeleton/src/clients/clipboard/copy.rs:25`) looks up the `clipboard`
service with `lookup_port(b"clipboard")` (`userland/app_skeleton/src/discover/lookup.rs:19`, backed by
`mk_service_lookup`), builds a payload of `CONTENT_TYPE_TEXT` plus the bytes, and sends `OP_COPY` through
`call_status` (`userland/app_skeleton/src/wire/call.rs:25`, backed by `mk_ipc_call`). A nonzero status
surfaces as `"clipboard rejected copy"`, and a missing service as `"clipboard not available"`.

`clipboard_paste` (`userland/app_skeleton/src/clients/clipboard/paste.rs:26`) looks up the port the same
way, sends `OP_PASTE` with the text content type through `call_payload`
(`userland/app_skeleton/src/wire/call_payload.rs:23`), then reads the `len: u32` after the status and
copies out the smaller of the reported length, the available bytes, and the caller's buffer. An empty
clipboard returns `Ok(0)`.

Two real capsules drive these helpers today. The text editor copies its buffer on Ctrl+C
(`userland/capsule_text_editor/src/editor/ctrl_copy.rs:22`) and pastes on Ctrl+V, validating the pasted
bytes as UTF-8 before inserting (`userland/capsule_text_editor/src/editor/ctrl_paste.rs:25`). The terminal
copies the current line (`userland/capsule_terminal/src/event/copy_line.rs:22`) and pastes into its input,
keeping only printable ASCII in the `0x20..=0x7E` range
(`userland/capsule_terminal/src/event/paste_clipboard.rs:30`). Both treat a clipboard error as a
non-fatal, best-effort outcome rather than a crash.

## Failure modes

Every failure is a typed status on the reply, not a panic or a dropped connection:

```
  E_INVAL       = -22   payload too short for the op (copy/paste/history_get under 4 bytes,
                        set_idle_timeout under 8)
  E_RANGE       = -34   entry over 64 KiB on copy; index out of range on history_get;
                        reply would not fit the output buffer; idle timeout out of the 5s..24h band
  E_BAD_OP      = -38   op not in 1..7
  E_BAD_MAGIC   = -71   header magic not 0x43424930
  E_BAD_LEN     = -90   buffer shorter than the header, or shorter than header + declared payload_len
  E_BAD_VERSION = -93   header version not 1
```

These are defined in `src/protocol/errno.rs:17`. Beyond the wire, the only other failure is heap init at
startup, which exits with status `1` (`src/main.rs:14`). A receive that returns nothing or a zero sender
pid is not an error; the loop yields and retries (`src/server/runner.rs:42`). A reply of zero length is
never sent (`src/server/runner.rs:49`), though every handler returns at least a 24-byte status reply, so
that guard is defensive.

## Security analysis

A clipboard is a cross-capsule data channel, so its security posture is worth stating plainly. The mask is
`0x19` (`CAPSULE_REQUIRED_CAPS` in `userland/capsule_clipboard/Capsule.mk:13`), which decodes to
`CoreExec | IPC | Memory` against `src/capabilities/types.rs:56` (CoreExec = 1, IPC = 8, Memory = 16, sum
`0x19`). That is the minimum a service leaf needs: run, receive and reply over IPC, and hold a heap. It
has no graphics, no filesystem, no network, no hardware, no crypto, and no debug. The kernel mirror asserts
the same value: `REQUIRED_CAPS = 0x19` in `src/userspace/capsule_clipboard/spawn.rs:35`. The clipboard
never touches the screen, never persists to disk, and never reaches a device, so the whole capsule is a
byte store behind an IPC port.

- No persistence. The mask carries no `FileSystem`, and the store is a `VecDeque` in RAM
  (`src/state/clipboard/types.rs:21`). Copied content lives only in the capsule's heap and is gone when
  it exits, and the idle wipe (`src/state/clipboard/timer.rs:26`) empties the whole history after the
  timeout even while the capsule keeps running.
- Bounded, so it cannot be flooded. `OP_COPY` (`src/server/handlers/copy.rs:21`) rejects any single entry
  over 64 KiB and the storage layer (`src/state/clipboard/storage.rs:24`) evicts the oldest entry when
  either the 16-entry depth or the 256 KiB total is exceeded, so a caller cannot grow the store without
  bound or exhaust the heap with one paste.
- No per-caller isolation. This is the honest boundary. The store is system-wide and the handlers do not
  check `sender_pid` against any notion of entry ownership; the sender pid is used only to address the
  reply (`src/server/runner.rs:50`). Any capsule that can reach port 4414 can `OP_PASTE`,
  `OP_HISTORY_LIST`, or `OP_HISTORY_GET` whatever another capsule copied, and can `OP_CLEAR` another
  capsule's content or shorten the retention window with `OP_SET_IDLE_TIMEOUT`. The confidentiality of a
  copied secret rests on the idle wipe and on the fact that reaching the port at all requires the IPC
  capability (`.../install.rs:50` stamps `Capability::IPC` on the endpoint), not on per-entry ownership.
  The trust implication is direct: if a capsule can talk to the clipboard, it can read everything anyone
  else has copied since the last wipe. Callers that handle secrets should treat the clipboard as a shared,
  world-readable channel among all IPC-capable capsules and clear it (`OP_CLEAR`) or keep the retention
  window short rather than relying on the store to keep one caller's data from another.
- No content interpretation. Bytes go in and come back unchanged; the content type is an opaque tag and
  the capsule never parses the payload, so there is no format-confusion or injection surface inside the
  service itself. Callers do their own validation (the editor checks UTF-8, the terminal filters to
  printable ASCII).

The `debug_tag` in the kernel spawn spec is the empty string (`src/userspace/capsule_clipboard/spawn.rs:51`)
and the mask has no `Debug` bit, so the capsule emits no `MkDebug` markers. Nothing about a copied secret
reaches the serial log through this capsule.

## Debugging

The service binds directly to port 4414 with `mk_ipc_recv_from` (`src/server/runner.rs:24`,
`SERVICE_PORT`). A client reaches it by looking up `clipboard` in the service registry
(`mk_service_lookup`, wrapped by `lookup_port`) and sending to the port that resolves, which is 4414. The
kernel prints the spawn marker as the capsule comes up (`src/kernel_core/process_spawn/capsule_spawn/runner/install/spawn_log.rs:17`):

```
  [SPAWN] name=clipboard pid=0x... caps=0x19 entry=0x...
```

`caps=0x19` in that line confirms the capsule was admitted with exactly `CoreExec | IPC | Memory` and
nothing was added at load. The clipboard is also one of the names the spawn tracer prints extra
install-stage lines for (`src/kernel_core/process_spawn/capsule_spawn/runner/install/trace.rs:17`), so a
stall during its install shows up as a `[SPAWN] clipboard ...` stage line (for example
`[SPAWN] clipboard runqueue ok`) rather than silence.

Once up, the failure signatures are on the wire, not the console. A `COPY` of an oversized entry returns
`E_RANGE` (-34) rather than storing it; an op outside 1..7 returns `E_BAD_OP` (-38); a bad header returns
`E_BAD_MAGIC` (-71), `E_BAD_VERSION` (-93), or `E_BAD_LEN` (-90); and a caller that finds the history empty
after leaving the machine idle is seeing the idle wipe, not a crash. If a client's
`mk_service_lookup("clipboard")` returns a zero port or a zero pid, the service never registered (the
capsule failed to spawn), which the client surfaces as its own `"clipboard not available"` rather than an
error from the clipboard. Because the capsule emits no debug markers of its own, wire status codes and the
one `[SPAWN]` line are the whole observable surface.

## Source map

```
  userland/capsule_clipboard/src/main.rs                     _start, heap init, hand-off to run()
  userland/capsule_clipboard/src/server/runner.rs            the port-4414 loop and idle GC
  userland/capsule_clipboard/src/server/respond.rs           status and payload reply builders
  userland/capsule_clipboard/src/server/handlers/router.rs   op dispatch and E_BAD_OP
  userland/capsule_clipboard/src/server/handlers/copy.rs     bounds check + FIFO push
  userland/capsule_clipboard/src/server/handlers/paste.rs    latest-of-type recall
  userland/capsule_clipboard/src/server/handlers/history_list.rs   (content_type, len) pairs
  userland/capsule_clipboard/src/server/handlers/history_get.rs    entry by index
  userland/capsule_clipboard/src/server/handlers/clear.rs    wipe every entry
  userland/capsule_clipboard/src/server/handlers/health.rs   liveness ping
  userland/capsule_clipboard/src/server/handlers/set_idle_timeout.rs   idle bound 5s..24h, 0 disables
  userland/capsule_clipboard/src/protocol/header.rs          magic 0x43424930 = CBI0, version 1, HDR_LEN 20
  userland/capsule_clipboard/src/protocol/ops.rs             the seven opcodes
  userland/capsule_clipboard/src/protocol/limits.rs          depth 16, 256 KiB total, 64 KiB entry, timeouts
  userland/capsule_clipboard/src/protocol/errno.rs           the typed errno table
  userland/capsule_clipboard/src/protocol/decode.rs          bounds-checked LE request parse
  userland/capsule_clipboard/src/protocol/encode.rs          reply header + status writers
  userland/capsule_clipboard/src/state/entry.rs              content_type + Vec<u8> record
  userland/capsule_clipboard/src/state/clipboard/types.rs    the VecDeque, byte total, activity timestamp
  userland/capsule_clipboard/src/state/clipboard/storage.rs  push/evict, latest_of_type, get_by_index
  userland/capsule_clipboard/src/state/clipboard/timer.rs    the idle-wipe timer
  userland/capsule_clipboard/Capsule.mk                      CAPSULE_REQUIRED_CAPS = 0x19, endpoints 4414/4415
  userland/app_skeleton/src/clients/clipboard/               copy/paste client helpers over mk_ipc_call
  userland/capsule_text_editor/src/editor/                   Ctrl+C / Ctrl+V clients
  userland/capsule_terminal/src/event/                       copy-line / paste-clipboard clients
  src/userspace/capsule_clipboard/spawn.rs                   kernel spawn spec, caps 0x19, ports 4414/4415
  src/kernel_core/process_spawn/capsule_spawn/runner/install/   endpoint registration + spawn log + trace
  src/capabilities/types.rs                                  CoreExec|IPC|Memory bit values
```

Every reference above is verified against those trees.

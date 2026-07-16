# capsule_vfs

`capsule_vfs` is the filesystem service the rest of the desktop reads and writes files through: the
`vfs_pool`. It is a RAM-resident, flat-namespace store fronted by an IPC protocol, and it is the pool the
terminal, the file manager, and the text editor all open the same files in. It owns a store of files and
directories plus a per-caller file-descriptor table, and it serves a POSIX-shaped operation set with two
isolation boundaries built into every request: a caller cannot impersonate another pid, and a descriptor
cannot be used by any pid but the one that opened it. Its path canonicalization, caller attestation, and
protocol codec are the same code the [`fs_proofs`](../../subsystems/storage/vfs-and-paths.md) suite
machine-tests off-target.

It is a userland service capsule, not a GUI app. The kernel spawns it under service handle `vfs_pool` on
service port 4104 with a reply endpoint on 4105, and its capability mask is `0x19`
(`userland/capsule_vfs/Capsule.mk:15`). The source is `userland/capsule_vfs/`, mirrored into the kernel as
the trusted-path copy at `src/fs/vfs_capsule/` (`Capsule.mk:16`).

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [The wire protocol](#the-wire-protocol)
- [The operation table](#the-operation-table)
- [Operation reference](#operation-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [The server loop](#the-server-loop)
- [Caller attestation](#caller-attestation)
- [The store model](#the-store-model)
- [File-descriptor ownership](#file-descriptor-ownership)
- [The path model](#the-path-model)
- [The read-only guard](#the-read-only-guard)
- [Zeroization](#zeroization)
- [Error mapping](#error-mapping)
- [Protocol and clients](#protocol-and-clients)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## Overview

The vfs pool is a single-threaded request/reply service. It initializes a heap, seeds a small store, and
loops forever receiving IPC frames, decoding each into a request, dispatching it to a handler, and sending
one reply. There is no shared state between callers other than the store itself and the fd table, and every
handler is keyed on the kernel-attested sender pid rather than any value the sender put in the payload.

The store is flat and RAM-resident. A file's `name` is its full absolute path, directories are just files
with the directory flag set, and there is no inode tree, so listing is a prefix scan rather than a walk.
Nothing persists across a reboot; this is the application filesystem the desktop lives in, deliberately
volatile. Because the same source is compiled into the kernel as `src/fs/vfs_capsule/` and exercised by
`fs_proofs`, a protocol or path regression usually surfaces in the proof suite before it ships.

## Identity

Everything the kernel and the service registry need to name and reach the vfs pool comes from its
`Capsule.mk`. The values below are the file, not paraphrase.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `vfs` | `Capsule.mk:5` |
| Service handle | `vfs` | `Capsule.mk:6` |
| Binary name | `vfs` | `Capsule.mk:9` |
| Namespace | `systems.nonos.vfs` | `Capsule.mk:11` |
| Service endpoint | `service:4104:vfs_pool` | `Capsule.mk:12` |
| Reply endpoint | `reply:4105:endpoint.4294967301` | `Capsule.mk:13` |
| Capability mask | `0x19` | `Capsule.mk:15` |
| Kernel mirror | `src/fs/vfs_capsule` | `Capsule.mk:16` |

The service name callers resolve is `vfs_pool` (`userland/app_skeleton/src/clients/vfs/types.rs:17`), which
the kernel mirror registers on port 4104 (`src/fs/vfs_capsule/spawn.rs:29`, `spawn.rs:30`). Replies go to
the fixed kernel reply endpoint `0x1_0000_0005` (decimal 4294967301,
`userland/capsule_vfs/src/protocol/types.rs:48`), which the capsule sends every response to.

The mask `0x19` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x0001  CoreExec   bit()  1     types.rs:56
  0x0008  IPC        bit()  8     types.rs:59
  0x0010  Memory     bit() 16     types.rs:60
  ------
  0x0019  = 1 + 8 + 16
```

The `Capsule.mk` comment states the same decomposition and the reason: `CAP_VFS` (the filesystem
capability other capsules present to reach a filesystem) is the caller-facing gate, not this capsule's own
bit; the vfs pool itself only needs IPC for the `mk_ipc_*` calls and Memory for its heap
(`Capsule.mk:1`, `Capsule.mk:14`). There is no `FileSystem` bit (64), because it *is* the filesystem rather
than a client of one; no `Network` (4), no `Crypto` (32), no `Debug` (256); and nothing from the driver
broker family (`Driver` 65536, `Mmio` 131072, `Irq` 262144, `Dma` 524288, `Pio` 1048576,
`src/capabilities/types.rs:72`). That empty hardware grant is the whole basis of the security analysis
below: the most-fuzzed surface here (path handling and the protocol codec) cannot reach a device.

## The wire protocol

A request is a framed message (`src/protocol/types.rs:17`, `decode.rs:27`) with a 20-byte header
(`HDR_LEN`, `types.rs:50`) and a payload. The header is little-endian throughout:

```
  offset  field         size
  0       magic         u32   0x4E4F5646 ("NOVF")   types.rs:17
  4       version       u16   1                     types.rs:18
  6       op            u16                          decode.rs:39
  8       flags         u16                          decode.rs:40
  10      (reserved)    u16                          (skipped)
  12      request_id    u32                          decode.rs:41
  16      payload_len   u32   <= 65536               decode.rs:42
  20      payload       payload_len bytes            decode.rs:50
```

`decode_request` rejects a short buffer (`DecodeError::Short`), a wrong magic (`BadMagic`), a wrong version
(`BadVersion`), and a `payload_len` over `MAX_PAYLOAD_BYTES` or a buffer shorter than header plus payload
(`BadLength`) (`decode.rs:28`-`49`). Any decode error is answered `EINVAL` by the server loop rather than
dropped (`server/runner.rs:40`). The response is built by `encode_response` (`protocol/encode.rs:21`) with
the same header shape, a `payload_len` of `4 + body.len()`, then a `status` i32 (0 or a negative errno) at
offset 20, then the optional body.

The fixed bounds and flags (`types.rs:36`-`44`):

```
  MAGIC = 0x4E4F5646 ("NOVF")   VERSION = 1
  MAX_PATH_BYTES    = 256        MAX_DATA_BYTES  = 65536
  MAX_LIST_BYTES    = 65536      MAX_PAYLOAD_BYTES = 65536
  open flags:  O_CREATE = 1<<0   O_TRUNC = 1<<1   O_APPEND = 1<<2
  rmdir flag:  F_RECURSIVE = 1<<0   (in the request flags field, not the payload)
```

Every request payload begins with a four-byte little-endian caller pid (see
[caller attestation](#caller-attestation)); the layouts below are what follows that pid.

## The operation table

Fifteen ops are defined (`src/protocol/types.rs:20`-`34`), each dispatched by `dispatch`
(`src/server/dispatch.rs:27`) to one handler; an unrecognised op is answered `EINVAL`
(`dispatch.rs:44`), so the surface is exactly these fifteen.

```
  OP  name         request payload (after u32 caller_pid)         reply body
  1   OPEN         u8 path_len, path, u32 flags                   u32 fd
  2   CLOSE        u32 fd                                         (empty)
  3   READ         u32 fd, u32 max_bytes                          bytes read
  4   WRITE        u32 fd, data bytes                             u32 written
  5   STAT         u8 path_len, path                              u64 size, u32 flags, u64 mtime, u16 mode
  6   LIST         u8 prefix_len, prefix                          <u8 name_len><name> entries
  7   HEALTHCHECK  (none)                                         (empty)
  8   MKDIR        u8 path_len, path                              (empty)
  9   UNLINK       u8 path_len, path                              (empty)
  10  RENAME       u8 from_len, from, u8 to_len, to               (empty)
  11  COPY         u8 src_len, src, u8 dst_len, dst, u8 recursive (empty)
  12  RMDIR        u8 path_len, path (F_RECURSIVE via req flags)  (empty)
  13  TRUNCATE     u8 path_len, path, u64 size                    (empty)
  14  USAGE        (none)                                         u32 files, u64 bytes, u32 max_files
  15  CHMOD        u8 path_len, path, u16 mode                    (empty)
```

Every handler validates its fixed layout before touching the store: a zero or oversize path length, a short
payload, or non-UTF-8 path bytes is `EINVAL`. `open` is representative
(`src/server/handlers/open.rs:27`-`61`): it parses `path_len` (bounded at `MAX_PATH_BYTES`), the UTF-8
path, and the u32 flags, then calls `store.open(path, pid, create, truncate, append, true)` with the final
`true` requesting write intent.

## Operation reference

Each entry gives the opcode, request layout, reply, and every error the handler and its store call can
return.

**OPEN (1)** `open.rs:27`. Payload: `u32 caller_pid, u8 path_len, path, u32 flags`. Flags `O_CREATE`,
`O_TRUNC`, `O_APPEND` (`open.rs:54`-`56`). Reply body: `u32 fd`. The handler returns `EINVAL` on an empty
rest, a zero or over-256 `path_len`, a payload too short for the path plus the 4 flag bytes, or non-UTF-8
(`open.rs:32`-`46`). The store (`store/fdtable/open.rs:23`) creates the file only if `O_CREATE` is set,
else `NotFound` (`ENOENT`); a directory path is `IsDir` (`EISDIR`); it computes
`writable = write && (mode & 0o200 != 0)`, so opening a non-owner-writable file for write yields no write
permission, and `O_TRUNC` without a writable file is `AccessDenied` (`EACCES`, `open.rs:40`); a full fd
table is `Full` (`ENOSPC`, `open.rs:48`); a new file over the 2048 ceiling is `Full` (`open.rs:58`). A new
file is created mode `0o644` (`open.rs:65`).

**CLOSE (2)** `close.rs:24`. Payload: `u32 caller_pid, u32 fd`. Reply: empty. `EINVAL` if the rest is not
exactly 4 bytes. The store `close` (`store/fdtable/close.rs:20`) frees the slot only through
`slot_mut`, which returns `BadFd` (`EBADF`) if the fd is out of range or not owned by this pid.

**READ (3)** `read.rs:24`. Payload: `u32 caller_pid, u32 fd, u32 max_bytes`. Reply body: the bytes read.
`EINVAL` if the rest is not exactly 8 bytes; `EMSGSIZE` if `max_bytes > MAX_DATA_BYTES`
(`read.rs:34`). The store `read` (`store/fdtable/read.rs:22`) resolves the fd for its owner (`BadFd`
otherwise), copies at most `max_bytes` from the current position, and advances the position; end of file
returns fewer bytes, not an error.

**WRITE (4)** `write.rs:24`. Payload: `u32 caller_pid, u32 fd, data`. Reply body: `u32 written`. `EINVAL`
if the rest is under 4 bytes; `EMSGSIZE` if `data.len() > MAX_DATA_BYTES` (`write.rs:34`). The store
`write` (`store/fdtable/write.rs:20`) requires the fd's `writable` flag (`AccessDenied` otherwise), writes
at `pos` or, for an append fd, at end of file, grows the file with a zero-filled gap if needed, rejects a
write past the 1 MiB ceiling as `Full`, and advances the position.

**STAT (5)** `stat.rs:26`. Payload: `u32 caller_pid, u8 path_len, path`. Reply body:
`u64 size, u32 flags, u64 mtime, u16 mode` (22 bytes; `flags` is 1 for a directory, else 0, `stat.rs:49`).
`EINVAL` on the usual length or UTF-8 failures; `NotFound` (`ENOENT`) if the path is absent. For a directory
the reported `size` is the count of immediate children, not a byte length (`store/fdtable/query.rs:26`).

**LIST (6)** `list.rs:29`. Payload: `u32 caller_pid, u8 prefix_len, prefix`. Reply body: concatenated
`<u8 name_len><name bytes>` entries, each directory name suffixed with `/`, capped at `MAX_LIST_BYTES` and
skipping any single name over 255 bytes (`store/fdtable/query.rs:47`). `EINVAL` on an empty rest, an
over-256 prefix, a truncated payload, or non-UTF-8. This is a raw prefix match over the flat namespace, so
the caller gets every stored path that begins with the prefix, not one directory level.

**HEALTHCHECK (7)** `healthcheck.rs:21`. No payload beyond the header is required; it echoes op, flags, and
request id with status 0 and an empty body. It does not call `split_caller`, so it is the one op that never
attests a caller.

**MKDIR (8)** `mkdir.rs:24`. Payload: `u32 caller_pid, u8 path_len, path`. Reply: empty. `EINVAL` on the
length or UTF-8 checks. The store `mkdir` (`store/fdtable/mkdir.rs:23`) returns `Exists` (`EEXIST`) if the
exact path exists, creates any missing parent components as mode `0o755` directories, and returns `Full`
(`ENOSPC`) if a create would pass the 2048 ceiling.

**UNLINK (9)** `unlink.rs:24`. Payload: `u32 caller_pid, u8 path_len, path`. Reply: empty. `EINVAL` on the
usual checks. The store `unlink` (`store/fdtable/unlink.rs:22`) returns `NotFound` if absent, `NotEmpty`
(`ENOTEMPTY`) if the path is a non-empty directory, closes any open fds pointing at it, zeroizes its data,
and removes it.

**RENAME (10)** `rename.rs:24`. Payload: `u32 caller_pid, u8 from_len, from, u8 to_len, to`. Reply: empty.
`EINVAL` if either length is zero, over 256, or truncates the payload, or on non-UTF-8. The store `rename`
(`store/fdtable/rename.rs:22`) returns `NotFound` if the source is absent, `Exists` if the destination
exists, and for a directory rewrites every descendant's path prefix from `old/` to `new/`.

**COPY (11)** `copy.rs:26`. Payload: `u32 caller_pid, u8 src_len, src, u8 dst_len, dst, u8 recursive`
(the recursive byte is optional; absent or zero means non-recursive, `copy.rs:51`). Reply: empty. `EINVAL`
on the length or UTF-8 checks. The store `copy` (`store/fdtable/copy.rs:28`) returns `NotFound` if the
source is absent, `Exists` if the destination or any recursive descendant target exists, and `Full` if the
additions would pass the 2048 ceiling; a non-recursive directory copy clones only the directory entry.

**RMDIR (12)** `rmdir.rs:27`. Payload: `u32 caller_pid, u8 path_len, path`; the `F_RECURSIVE` bit lives in
the request *flags* field, not the payload (`rmdir.rs:43`). Reply: empty. `EINVAL` on the length or UTF-8
checks. The store `rmdir` (`store/fdtable/rmdir.rs:27`) returns `NotFound` if absent, `NotEmpty` if the
directory has children and `F_RECURSIVE` is clear, and otherwise removes the directory and its whole
subtree, closing open fds, reindexing survivors, and zeroizing removed data.

**TRUNCATE (13)** `truncate.rs:26`. Payload: `u32 caller_pid, u8 path_len, path, u64 size`. Reply: empty.
`EINVAL` on the length or UTF-8 checks. This handler normalizes the path and applies the read-only guard:
`EACCES` if the normalized path is under `/capsules` (`truncate.rs:45`-`48`). The store `truncate`
(`store/fdtable/truncate.rs:22`) returns `NotFound` if absent, `IsDir` for a directory, `Full` if `size`
exceeds 1 MiB, zeroizes the dropped tail on a shrink, and zero-fills on a grow.

**USAGE (14)** `usage.rs:25`. Payload: `u32 caller_pid`. Reply body: `u32 file_count, u64 bytes_used,
u32 max_files` (`store/fdtable/usage.rs:22`). The only error path is the attestation failure returned by
`split_caller`.

**CHMOD (15)** `chmod.rs:26`. Payload: `u32 caller_pid, u8 path_len, path, u16 mode`. Reply: empty. `EINVAL`
on the length or UTF-8 checks. Like truncate, it normalizes and applies the read-only guard: `EACCES` under
`/capsules` (`chmod.rs:43`-`46`). The store `chmod` (`store/fdtable/chmod.rs:21`) returns `NotFound` if
absent and otherwise sets `mode & 0o777`.

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` initializes the heap (`mk_exit(1)` on failure) and calls
`server::run`, which never returns (`src/main.rs:29`-`34`). The three top-level modules are `protocol` (the
NOVF frame, ops, flags, bounds, codec), `server` (the loop, the dispatcher, the handlers, path handling),
and `store` (the flat store and fd table) (`src/main.rs:22`-`24`).

Lifecycle:

1. The kernel spawns the mirror as one of the first capsules through the boot plan: `spawn_vfs`
   (`src/userspace/init/spawn_plan/core.rs:29`) calls `boot::capsule("VFS", "vfs", ...)` (`core.rs:32`),
   which runs `capsule_boot::boot` (`spawn_plan/boot.rs:26`). On success it logs `[VFS] capsule spawned`
   and registers the capsule with the lifecycle table
   (`src/userspace/init/capsule_boot/run.rs:29`-`30`); on failure it logs an `[ERROR]` line with the
   `SpawnError` (`run.rs:32`).
2. `server::run` allocates a 65556-byte buffer (64 KiB payload plus header slack), builds the store, and
   seeds it (`server/runner.rs:27`-`30`).
3. The store seed creates `/docs` and an empty `/capsules`, and three files `/readme.txt`,
   `/docs/about.txt`, `/docs/demo.txt` (`store/fdtable/seed.rs:27`-`37`). `/capsules` starts empty on
   purpose: the installer lands verified capsule artifacts there at runtime rather than baking them into the
   image (`seed.rs:29`-`33`).
4. The loop receives one frame at a time, decodes, dispatches, and replies, forever
   (`server/runner.rs:31`-`43`).

The store is two vectors: `files` and `fds`, preallocated to 256 `None` slots at construction
(`store/fdtable/new.rs:22`-`28`). File timestamps come from `mk_time_millis`, clamped to 0 before the
kernel clock is up so a file carries no known timestamp rather than a garbage one
(`store/fdtable/time.rs:22`-`29`).

## The server loop

The loop (`src/server/runner.rs:27`) is the shape every service capsule shares:

```
  run():
      buf   = vec![0; 65556]              // 64 KiB payload + header slack
      store = Store::new(); store.seed()
      loop:
          sender_pid = 0
          n = mk_ipc_recv_from(inbox 0, buf, MAX_MSG, 0, &sender_pid)   // kernel stamps sender_pid
          if n <= 0: continue
          resp = match decode_request(buf[..n]):
              Ok(req)  => dispatch(store, req, sender_pid)
              Err(_)   => encode_response(0, 0, 0, EINVAL, &[])
          mk_ipc_send(KERNEL_REPLY_ENDPOINT, resp)
```

`sender_pid` is the load-bearing argument: `mk_ipc_recv_from` returns the pid the kernel attested for the
message, not a value the sender chose, and that attested pid is threaded into every handler
(`runner.rs:33`, `runner.rs:39`). A frame that fails to decode is answered `EINVAL` rather than dropped, so
a malformed request does not stall the caller (`runner.rs:40`).

## Caller attestation

Every handler except healthcheck runs `split_caller` (`src/server/handlers/util.rs:47`) before anything
else, and `resolve_caller` (`util.rs:32`) is the no-impersonation rule:

```
  resolve_caller(payload_pid, sender_pid):
      if sender_pid == 0:            payload_pid    // the kernel-side mirror is the TCB, trusted
      if payload_pid == sender_pid:  sender_pid     // a ring-3 caller must match its attested pid
      else:                          None -> EACCES // impersonation attempt
```

A ring-3 capsule's message carries the kernel-attested `sender_pid`, and the handler requires the payload's
claimed pid to equal it (`util.rs:36`); a mismatch is `EACCES` (`util.rs:52`-`54`). The one exception is
the kernel-side mirror (`sender_pid == 0`), the trusted computing base, which keeps the payload pid
(`util.rs:33`). A payload shorter than 4 bytes is `EINVAL` before the check (`util.rs:48`). This exact
property is fuzzed by [`fs_proofs`](../../subsystems/storage/vfs-and-paths.md).

## The store model

The `Store` (`src/store/fdtable/types.rs:53`) is two vectors, files and open descriptors:

```
  struct Store { files: Vec<File>, fds: Vec<Option<OpenFd>> }

  struct File   { name: String, data: Vec<u8>, is_dir: bool, mode: u16, mtime: u64 }   // types.rs:37
  struct OpenFd { file_idx: usize, owner_pid: u32, pos: usize, append: bool, writable: bool }  // :45

  MAX_FILES = 2048    MAX_OPEN_FDS = 256    MAX_FILE_BYTES = 1 << 20 (1 MiB)   // types.rs:20-22
```

The namespace is flat: a `File`'s `name` is its full absolute path, and directories are `File`s with
`is_dir` set, so `find` is an exact-name search (`store/fdtable/lookup.rs:20`) and `list` filters by path
prefix rather than walking a tree. A file is at most one mebibyte, there are at most 2048 files, and at most
256 descriptors open at once; exceeding any is `StoreError::Full`, mapped to `ENOSPC`.

## File-descriptor ownership

A descriptor is bound to the pid that opened it. `entry` and `slot_mut`
(`src/store/fdtable/lookup.rs:24`, `:36`) resolve an fd only for its owner:

```
  entry(fd, owner_pid):
      if fd >= fds.len():                       BadFd
      match fds[fd]:
          Some(e) if e.owner_pid == owner_pid:  Ok(e)
          Some(_):                              BadFd     // wrong owner
          None:                                 BadFd     // empty slot
```

So a capsule cannot read or write through a descriptor another capsule opened, even by guessing the fd
number, because the store checks the attested owner pid on every access, and a wrong owner is
indistinguishable from a bad fd (`BadFd` -> `EBADF`). This is the second isolation boundary after caller
attestation: attestation stops impersonation at the request, fd ownership stops it at the descriptor.

## The path model

Paths are absolute strings that name a whole entry; there is no per-caller current directory in the vfs
pool (the shell's `cwd` lives in the terminal, and the terminal resolves paths to absolute before it calls).
The canonicalizer `normalize_to_buffer` (`src/server/handlers/path/normalize_to_buffer.rs:17`) folds a raw
path into a single leading slash form: it collapses runs of `/`, drops empty and `.` components, and pops
one component per `..` without ever escaping the root. `normalize` wraps it into an owned `String`
(`path/normalize.rs:22`). This is the same routine the kernel mirror re-exports as `pub(crate)`
(`path/mod.rs:23`), which is why the path tests in `fs_proofs` exercise the exact production code.

The important subtlety is where normalization actually runs. Only the `truncate` and `chmod` handlers call
`normalize` before acting (`truncate.rs:45`, `chmod.rs:43`); those are the two ops that carry the read-only
guard. Every other op (`open`, `read`, `write`, `stat`, `list`, `mkdir`, `unlink`, `rename`, `copy`,
`rmdir`) passes the caller's path bytes straight to the store, where `find` does an exact string compare.
So callers are expected to send already-canonical absolute paths, which the app-skeleton vfs client and the
terminal's `cwd::resolve` do, and the store's flat-name compare means a non-canonical path simply does not
match any stored entry rather than reaching a different one. The [honest gaps](#honest-gaps) note this
plainly.

## The read-only guard

The signed-capsule tree is protected on the two ops that can rewrite an existing file's mode or length.
`is_read_only` (`src/server/handlers/path/is_read_only.rs:17`) treats `/capsules` and everything under it
as non-writable:

```
  is_read_only(path):  path == "/capsules" || path.starts_with("/capsules/")
```

Because `chmod` and `truncate` normalize first (`chmod.rs:43`, `truncate.rs:45`), the guard runs on the
canonical path, so a trailing slash (`/capsules//x`) or a traversal round-trip (`/capsules/../capsules/x`)
both normalize back under `/capsules` and are refused with `EACCES`. `fs_proofs` proves those bypasses
fail. The honest limit, stated in the gaps section, is that the guard is not wired into `write`, `unlink`,
`rename`, or `rmdir`, so it is a chmod/truncate protection rather than a blanket write barrier on
`/capsules`; in practice the tree is seeded empty and the installer, not a client, populates it.

## Zeroization

When a file's data is freed or truncated, it is zeroed first (`src/store/fdtable/zeroize.rs:22`):

```
  zeroize(buf):
      for byte in buf:  write_volatile(byte, 0)
      compiler_fence(SeqCst)
```

The volatile write plus the fence stops the optimizer from eliding the erase as a dead store, so a deleted
or truncated file's contents cannot linger in reclaimed heap to be read back by a later allocation. It runs
on unlink (`unlink.rs:38`), on the removed subtree in rmdir (`rmdir.rs:51`), on the dropped tail in truncate
(`truncate.rs:33`), and on `O_TRUNC` at open (`open.rs:44`). This is the per-file half of the
[ZeroState](../../subsystems/memory/zeroization.md) posture, mirroring the kernel [ramfs](ramfs.md)'s
zero-on-drop.

## Error mapping

`map_store_err` (`src/server/handlers/util.rs:20`) maps the store's errors to POSIX errnos
(`src/protocol/errno.rs`), so a caller sees familiar codes:

```
  NotFound -> ENOENT (-2)    BadFd -> EBADF (-9)     Full -> ENOSPC (-28)
  AccessDenied -> EACCES (-13)   Exists -> EEXIST (-17)
  NotEmpty -> ENOTEMPTY (-39)    IsDir -> EISDIR (-21)
```

Two errnos are returned directly by handlers rather than through the store: `EINVAL` (-22) for a malformed
frame or payload, and `EMSGSIZE` (-90) for a read or write whose byte count exceeds `MAX_DATA_BYTES`
(`read.rs:35`, `write.rs:35`, `errno.rs:25`).

## Protocol and clients

The vfs pool speaks only its own NOVF protocol on the `vfs_pool` service; it makes no outbound IPC calls of
its own. Its clients reach it through the app-skeleton vfs client, whose op constants mirror the server's
one for one (`userland/app_skeleton/src/clients/vfs/types.rs:17`-`32`): `OP_OPEN 1`, `OP_CLOSE 2`,
`OP_READ 3`, `OP_WRITE 4`, `OP_STAT 5`, `OP_LIST 6`, `OP_MKDIR 8`, `OP_UNLINK 9`, `OP_RENAME 10`,
`OP_COPY 11`, `OP_RMDIR 12`, `OP_TRUNCATE 13`, `OP_USAGE 14`, `OP_CHMOD 15`, plus `O_CREATE`/`O_TRUNC`, all
against magic `0x4E4F5646` and service name `vfs_pool`.

The clients in the tree:

- The terminal, for every filesystem verb: `ls`, `cat`/`read`, `write`, `touch`, `mk`/`mkdir`, `mv`, `rm`,
  `stat`, `find`, `du`, and tab completion, plus `> >> <` redirection, all resolve against the shell's cwd
  and call the vfs client (for example `userland/capsule_terminal/src/command/builtin/nox/ls.rs`,
  `.../stat.rs`, `.../mv.rs`, `.../rm.rs`, `.../mk.rs`, `.../touch.rs`,
  `.../command/dispatch/run.rs`, `.../event/on_tab.rs`).
- The file manager, for its directory listing, metadata, previews, and permission display
  (`userland/capsule_file_manager/src/fm/refresh_meta.rs`, `.../preview.rs`, `.../perms.rs`).
- The text editor, for open and save (`userland/capsule_text_editor/src/editor/ctrl_open.rs`,
  `.../ctrl_save.rs`).
- The sovereign std filesystem session layer (`userland/sdk/nonos_std/src/fs/session.rs`).

Because all of them read and write the same pool, a file the terminal writes appears in the file manager and
opens in the editor, which is exactly the demo the seed `/docs/demo.txt` describes (`seed.rs:24`).

## Security analysis

The vfs pool's guarantees stack:

- **Caller attestation** stops one capsule from issuing a request as another; the pid is the kernel's
  attested `sender_pid`, not a payload field (`util.rs:47`).
- **Descriptor ownership** stops one capsule from using another's open fd, checked on every access
  (`lookup.rs:24`).
- **Write permission** is enforced at open: a file is writable only if the caller asked for write and the
  file's owner-write bit is set (`open.rs:39`), and a write through a non-writable fd is `AccessDenied`
  (`write.rs:25`).
- **The read-only guard** stops `chmod` and `truncate` from touching the signed `/capsules` tree, on the
  normalized path so it cannot be tricked (`chmod.rs:44`, `truncate.rs:46`).
- **The bounds** (path 256 bytes, data 64 KiB per call, file 1 MiB, 2048 files, 256 fds) stop a caller from
  exhausting the store through one oversize request or a flood.
- **Zeroization** stops a deleted or shrunk file's bytes from lingering (`zeroize.rs:22`).

All of these run against this exact source in `fs_proofs`, which pulls the handler, path, store, and
protocol code in and fuzzes normalization, the read-only guard, the codec, and caller attestation
(`userland/fs_proofs/src/vfs_path_tests.rs`, `.../protocol_tests.rs`, `.../store_tests.rs`,
`.../fuzz_tests.rs`, `.../kani_proofs.rs`). This is the one filesystem in the system whose access rules are
machine-verified.

The capability grant is the whole story. Mask `0x19` is CoreExec, IPC, and Memory and nothing else
(`Capsule.mk:15`, `src/capabilities/types.rs:56`,`:59`,`:60`). The filesystem service holds no hardware
capability (no Driver, Mmio, Irq, Dma, or Pio), so a bug in the most-fuzzed surface here, path handling or
the protocol codec, cannot reach a device, program DMA, or take an interrupt. It holds no `FileSystem` cap,
because it *is* the RAM filesystem rather than a client of a lower storage surface; no `Crypto`, because
unlike the encrypted [ramfs](ramfs.md) it stores plaintext and relies on zeroization rather than at-rest
encryption; no `Debug`, so it cannot open a serial surface; and no `RegisterService` beyond the one endpoint
its manifest declares. Its isolation from other capsules is the caller-attestation and fd-ownership pair
above, both keyed on the kernel-attested `sender_pid`.

## How to contribute

The source lives at `userland/capsule_vfs/`. The protocol is under `src/protocol/`, the request loop and
handlers under `src/server/`, and the store under `src/store/fdtable/`. The kernel-side mirror is
`src/fs/vfs_capsule/`; it compiles the same crate as the trusted-path copy, and `fs_proofs` tests both.

To add an operation:

1. Assign the next opcode in `src/protocol/types.rs:20` and document its request and reply layout in the
   comment above the handler.
2. Write the handler as one file under `src/server/handlers/` (for example `stat.rs`), calling
   `split_caller` first, validating the fixed layout to `EINVAL` before touching the store, and mapping
   store errors through `map_store_err`. If it mutates an existing entry and should honour the `/capsules`
   guard, `normalize` the path and check `is_read_only` the way `chmod.rs` and `truncate.rs` do.
3. Re-export the handler from `src/server/handlers/mod.rs` and add its arm to the match in
   `src/server/dispatch.rs:27`.
4. Add the store method as one file under `src/store/fdtable/` and re-export it from
   `src/store/fdtable.rs`, returning a `StoreError` variant that already maps to an errno, or extend
   `StoreError` (`store/fdtable/types.rs:24`) and `map_store_err` together.
5. Mirror the op constant into the app-skeleton client
   (`userland/app_skeleton/src/clients/vfs/types.rs`) and add a case to `fs_proofs`.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_vfs/Capsule.mk:18`):

```
  make nonos-mk-vfs               build the capsule ELF
  make nonos-mk-vfs-sign          produce the id cert, manifest, and attestation trailer
  make nonos-mk-vfs-verify        verify the signed manifest against the trust anchor policy
  make nonos-mk-check-vfs-keys    check the per-capsule ed25519 and ML-DSA signing keys exist
```

For a running kernel that includes the vfs pool, `make nonos-mk-vfs-prod` builds the vfs-only kernel
profile (`Makefile:925`), and `make nonos-mk-boot-vfs` runs the boot round-trip test
(`Makefile:1393`, `tests/boot/vfs_round_trip.sh`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns errors as encoded negative-errno responses, never a panic);
the one `unsafe` block, the volatile zeroize, carries its safety justification (`zeroize.rs:24`); modular
files, one unit per file, with `mod.rs`/`fdtable.rs` used only for re-exports; and the AGPL header at the
top of every source file, matching the header on every existing module (`src/main.rs:1`-`15`).

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints `[VFS] capsule
spawned` (`src/userspace/init/capsule_boot/run.rs:29`, formatted by `src/sys/boot_log/output.rs:33`). An
absent line means the capsule never started, usually a signature, manifest, or capability failure; the
error path prints an `[ERROR]` line instead (`run.rs:32`).

Because so much of the desktop reads files, a missing `[VFS]` marker cascades: apps that resolve
`vfs_pool` get no pid and every open fails, so a boot that reaches the desktop but where nothing can read a
file is a vfs-did-not-register symptom, not an app bug. Once registered, the failure signatures are the
errnos from `map_store_err`:

- `EACCES` from attestation means a payload pid that did not match the attested sender (an impersonation
  attempt); `EACCES` from `chmod`/`truncate` means the path normalized under `/capsules`.
- `EBADF` means an fd used by a pid that did not open it, or an out-of-range fd.
- `EACCES` from `write` means the fd was opened without write permission, which for an existing file means
  its owner-write bit was clear at open time.
- `ENOSPC` means one of the bounds was hit: 2048 files, 256 open fds, or 1 MiB in a single file.
- `EMSGSIZE` means a `read`/`write` asked for more than 64 KiB in one call.
- `EINVAL` means a malformed frame, a bad path length, or non-UTF-8 path bytes.

Because the kernel mirror runs the same code (`src/fs/vfs_capsule/`), the same protocol is exercised by
`fs_proofs` off-target, so a protocol or path regression usually shows up in the proof suite before it
ships.

## Honest gaps

Stated plainly:

- The namespace is flat (a file's name is its full path; there is no inode tree), so `list` is a raw prefix
  scan and returns every stored path under the prefix, not one directory level
  (`store/fdtable/query.rs:47`).
- Only `chmod` and `truncate` normalize the path and apply the `/capsules` read-only guard; the other ops
  pass the caller's path straight through and rely on callers sending canonical absolute paths, with the
  store's exact-name compare meaning a non-canonical path fails to match rather than reaching a different
  entry (`store/fdtable/lookup.rs:20`).
- `mode` is a real permission for one thing only, the owner-write bit checked at open (`open.rs:39`); it is
  not a full user/group/other model, and there is no ownership check on stat, read of a mode-000 file's
  metadata, or list.
- There are no symlinks and no file locks.
- There is no persistence: the store is in RAM and vanishes on reboot, which is the point; this is the
  RAM-resident application filesystem. The on-disk store for code that wants persistence is the dormant
  [blockfs](../../subsystems/storage/vfs-and-paths.md), not this capsule.

## Source map

```
  userland/capsule_vfs/src/main.rs                    heap init + server::run
  userland/capsule_vfs/src/protocol/types.rs          NOVF frame, ops, flags, bounds, reply endpoint
  userland/capsule_vfs/src/protocol/decode.rs         decode_request (magic/version/length checks)
  userland/capsule_vfs/src/protocol/encode.rs         encode_response (header + status + body)
  userland/capsule_vfs/src/protocol/errno.rs          the POSIX errno constants
  userland/capsule_vfs/src/server/runner.rs           the recv/decode/dispatch/reply loop
  userland/capsule_vfs/src/server/dispatch.rs         op -> handler (unknown -> EINVAL)
  userland/capsule_vfs/src/server/handlers/           the 15 handlers + util.rs (split_caller, map_store_err)
  userland/capsule_vfs/src/server/handlers/path/      normalize, normalize_to_buffer, is_read_only
  userland/capsule_vfs/src/store/fdtable/types.rs     Store, File, OpenFd, StoreError, the bounds
  userland/capsule_vfs/src/store/fdtable/             open/read/write/lookup/rename/copy/rmdir/chmod/...
  userland/capsule_vfs/src/store/fdtable/seed.rs      the seeded /docs, /capsules, and starter files
  userland/capsule_vfs/src/store/fdtable/zeroize.rs   the volatile erase
  userland/capsule_vfs/Capsule.mk                     slug, handle, ports, capability mask, kernel mirror
  src/fs/vfs_capsule/                                 the kernel-side mirror and verified spawn
  src/userspace/init/spawn_plan/core.rs               spawn_vfs -> boot::capsule("VFS", "vfs", ...)
  userland/app_skeleton/src/clients/vfs/              the vfs client the terminal, fm, and editor call
  userland/fs_proofs/                                 the machine-checked proofs of this source
  nonos-mk/capsule.mk                                 the generated nonos-mk-vfs[-sign|-verify] targets
```

Every reference above is verified against those trees.
</content>

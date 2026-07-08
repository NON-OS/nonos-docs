# capsule_vfs

`capsule_vfs` is the filesystem service the whole system talks to: the `vfs_pool`. It owns a store of
files and directories and a per-caller file-descriptor table, and it serves the full POSIX-shaped
operation set over IPC with three properties built into every request, a caller cannot impersonate
another, a descriptor cannot be used by anyone but the pid that opened it, and the signed-capsule tree is
read-only. Its path canonicalization, caller attestation, and protocol codec are the exact code the
[`fs_proofs`](../../subsystems/storage/vfs-and-paths.md) suite machine-tests. Service `vfs_pool` on port
4104, reply endpoint 4105, capability mask `0x19`. The source is `userland/capsule_vfs/`, mirrored into
the kernel at `src/userspace/capsule_vfs/`.

## Contents

- [The server loop](#the-server-loop)
- [The wire protocol](#the-wire-protocol)
- [The operation table](#the-operation-table)
- [Caller attestation](#caller-attestation)
- [The store model](#the-store-model)
- [File-descriptor ownership](#file-descriptor-ownership)
- [The write path in detail](#the-write-path-in-detail)
- [Path canonicalization and the read-only guard](#path-canonicalization-and-the-read-only-guard)
- [Zeroization](#zeroization)
- [Error mapping](#error-mapping)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop

The entry (`src/main.rs:29`) initializes the heap and calls `server::run`. The loop
(`src/server/runner.rs:27`) is the shape every service capsule shares, with a 64 KiB-plus buffer:

```
  run():
      buf   = vec![0; MAX_MSG]            // 65556 = 64 KiB payload + header slack
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

`sender_pid` is the load-bearing argument: `mk_ipc_recv_from` returns the pid the **kernel attested** for
the message, not a value the sender chose, and that attested pid is threaded into every handler. A frame
that fails to decode is answered `EINVAL` rather than dropped, so a malformed request does not stall the
caller.

## The wire protocol

A request is a framed message (`src/protocol/types.rs:17`) with magic `0x4E4F5646` ("NOVF"), version 1,
an operation, flags, a request id, and a payload. The bounds are fixed:

```
  MAGIC = 0x4E4F5646 ("NOVF")   VERSION = 1
  MAX_PATH_BYTES = 256          MAX_DATA_BYTES  = 65536
  MAX_LIST_BYTES = 65536        MAX_PAYLOAD_BYTES = 65536
  open flags:   O_CREATE = 1<<0   O_TRUNC = 1<<1   O_APPEND = 1<<2
  dir flags:    F_RECURSIVE = 1<<0
```

Every payload begins with a four-byte caller pid (see [caller attestation](#caller-attestation)), and the
response carries the op, flags, request id, a status (0 or a negative errno), and an optional body.

## The operation table

Fifteen operations (`src/protocol/types.rs:20`), each with a fixed payload layout after the caller pid:

```
  OP  name         request payload (after u32 caller_pid)              reply body
  1   OPEN         u8 path_len, path, u32 flags                        u32 fd
  2   CLOSE        u32 fd                                              -
  3   READ         u32 fd, u32 len                                     bytes
  4   WRITE        u32 fd, bytes                                       u32 written
  5   STAT         u8 path_len, path                                   size, mode, mtime, is_dir
  6   LIST         u8 prefix_len, prefix                               <u8 name_len><name> entries
  7   HEALTHCHECK  -                                                   -
  8   MKDIR        u8 path_len, path                                   -
  9   UNLINK       u8 path_len, path                                   -
  10  RENAME       u8 from_len, from, u8 to_len, to                    -
  11  COPY         u8 from_len, from, u8 to_len, to                    -
  12  RMDIR        u8 path_len, path (F_RECURSIVE via flags)           -
  13  TRUNCATE     u8 path_len, path, u64 size                         -
  14  USAGE        -                                                   used, capacity
  15  CHMOD        u8 path_len, path, u16 mode                         -
```

`dispatch` (`src/server/dispatch.rs:27`) is a match from op to handler; an unknown op is answered
`EINVAL`, so the protocol surface is exactly these fifteen. Every handler validates its fixed layout
before acting: a zero or oversize path length, a short payload, or non-UTF-8 path bytes is `EINVAL`
before the store is touched. `open` (`src/server/handlers/open.rs:27`) is representative, parsing
`path_len` (bounded at `MAX_PATH_BYTES`), the UTF-8 path, and the flags, then calling
`store.open(path, pid, create, truncate, append)`.

## Caller attestation

Every handler runs `split_caller` (`src/server/handlers/util.rs:47`) before anything else, and the rule
in `resolve_caller` is the no-impersonation guarantee:

```
  resolve_caller(payload_pid, sender_pid):
      if sender_pid == 0:            payload_pid    // the kernel-side mirror is the TCB, trusted
      if payload_pid == sender_pid:  sender_pid     // a ring-3 caller must match its attested pid
      else:                          None -> EACCES // impersonation attempt
```

A ring-3 capsule's message carries the kernel-attested `sender_pid`, and the handler requires the
payload's claimed pid to equal it. So a capsule cannot forge a request as another capsule to reach its
files; a mismatch is `EACCES`. The one exception is the kernel-side mirror (`sender_pid == 0`), the
trusted computing base, which keeps the payload pid. This exact property, that no sender can impersonate
another, is fuzzed by [`fs_proofs`](../../subsystems/storage/vfs-and-paths.md).

## The store model

The `Store` (`src/store/fdtable/types.rs:53`) is two vectors, files and open descriptors:

```
  struct Store { files: Vec<File>, fds: Vec<Option<OpenFd>> }

  struct File   { name: String, data: Vec<u8>, is_dir: bool, mode: u16, mtime: u64 }
  struct OpenFd { file_idx: usize, owner_pid: u32, pos: usize, append: bool, writable: bool }

  MAX_FILES = 2048    MAX_OPEN_FDS = 256    MAX_FILE_BYTES = 1 MiB
```

The namespace is flat: a `File`'s `name` is its full path, and directories are `File`s with `is_dir`
set, so `list` filters by path prefix rather than walking a tree. A file is at most one mebibyte, there
are at most 2048 files, and at most 256 descriptors open at once; exceeding any is `StoreError::Full`
mapped to `ENOSPC`.

## File-descriptor ownership

A descriptor is bound to the pid that opened it. `entry` and `slot_mut` (`src/store/fdtable/lookup.rs:24`)
resolve an fd only for its owner:

```
  entry(fd, owner_pid):
      if fd >= fds.len():                       BadFd
      match fds[fd]:
          Some(e) if e.owner_pid == owner_pid:  Ok(e)
          _:                                    BadFd     // wrong owner or empty slot
```

So a capsule cannot read or write through a descriptor another capsule opened, even by guessing the fd
number, because the store checks the attested owner pid on every access, and a mismatch is
indistinguishable from a bad fd (`BadFd` -> `EBADF`). This is the second isolation boundary after caller
attestation: attestation stops impersonation at the request, and fd ownership stops it at the descriptor.

## The write path in detail

`write` (`src/store/fdtable/write.rs:20`) shows the bounds and the append semantics:

```
  write(fd, owner_pid, bytes):
      (file_idx, append, pos, writable) = entry(fd, owner_pid)?
      if not writable:                   AccessDenied
      start = append ? data.len() : pos
      end   = start + bytes.len()        (saturating)
      if end > MAX_FILE_BYTES:           Full
      if end > data.len():               data.resize(end, 0)   // zero-fill any gap
      data[start..end] = bytes
      file.mtime = now_ms()
      fd.pos = end
```

A read-only descriptor (opened without write intent) is `AccessDenied`; a write past the one-mebibyte
ceiling is `Full`; a write past the current end grows the file with a zero-filled gap; and the
descriptor's position advances. Append ignores the position and always writes at the end.

## Path canonicalization and the read-only guard

Mutating operations normalize the path and refuse the signed-capsule tree. `chmod`
(`src/server/handlers/chmod.rs:43`) is representative:

```
  path = normalize(path)
  if is_read_only(path):  return EACCES
  store.chmod(path, mode)
```

`is_read_only` (`src/server/handlers/path/`) treats `/capsules` and everything under it as non-writable,
so a capsule cannot rewrite the signed images the system spawns from, and the guard runs on the
*normalized* path so it cannot be bypassed by a trailing slash (`/capsules//x`) or a traversal round-trip
(`/capsules/../capsules/x`), both of which normalize back under `/capsules`. `fs_proofs` proves those
bypasses fail. The normalization is the same canonicalization the [path defenses](../../subsystems/storage/vfs-and-paths.md)
page documents.

## Zeroization

When a file's data is freed or truncated, it is zeroed first (`src/store/fdtable/zeroize.rs:22`):

```
  zeroize(buf):
      for byte in buf:  write_volatile(byte, 0)
      compiler_fence(SeqCst)
```

The volatile write plus the fence stops the optimizer from eliding the erase as a dead store, so a
deleted or truncated file's contents cannot linger in reclaimed heap to be read back by a later
allocation. This is the per-file half of the [ZeroState](../../subsystems/memory/zeroization.md) posture,
mirroring the kernel [ramfs](ramfs.md)'s zero-on-drop.

## Error mapping

`map_store_err` (`src/server/handlers/util.rs:20`) maps the store's errors to POSIX errnos, so a caller
sees familiar codes:

```
  NotFound -> ENOENT    BadFd -> EBADF     Full -> ENOSPC     AccessDenied -> EACCES
  Exists   -> EEXIST    NotEmpty -> ENOTEMPTY   IsDir -> EISDIR
```

## Security analysis

The vfs pool's guarantees stack:

- **Caller attestation** stops one capsule from issuing a request as another; the pid is the kernel's
  attestation, not a payload field.
- **Descriptor ownership** stops one capsule from using another's open fd, checked on every access.
- **The read-only guard** stops any capsule from writing the signed `/capsules` tree, on the normalized
  path so it cannot be tricked.
- **The bounds** (path 256 bytes, file 1 MiB, 2048 files, 256 fds) stop a caller from exhausting the
  store through one request or a flood.
- **Zeroization** stops a deleted file's bytes from lingering.

All of these are tested against this exact source by `fs_proofs`, which includes the handler and path
code via `#[path]` and fuzzes normalization, the read-only guard, the protocol codec, and caller
attestation over millions of inputs. This is the one filesystem in the system whose access rules are
machine-verified.

## Honest gaps

Stated plainly: the namespace is flat (a file's name is its full path; there is no inode tree), so `list`
is a prefix scan; there are no symlinks; `mode` is stored but not enforced as a permission system beyond
the read-only guard and the per-descriptor writable flag; there are no file locks; and there is no
persistence, the store is in RAM (which is the point, this is the RAM-resident application filesystem).
The on-disk store for the code that wants persistence is the dormant [blockfs](../../subsystems/storage/vfs-and-paths.md),
not this capsule.

## Source map

```
  userland/capsule_vfs/src/main.rs                 heap init + server::run
  userland/capsule_vfs/src/server/runner.rs        the recv/decode/dispatch/reply loop
  userland/capsule_vfs/src/server/dispatch.rs      op -> handler (unknown -> EINVAL)
  userland/capsule_vfs/src/server/handlers/        the 15 handlers + util.rs (split_caller, map_store_err)
  userland/capsule_vfs/src/server/handlers/path/   normalize, is_read_only (the /capsules guard)
  userland/capsule_vfs/src/protocol/               the NOVF frame, ops, flags, bounds, encode/decode
  userland/capsule_vfs/src/store/fdtable/types.rs  Store, File, OpenFd, StoreError, the bounds
  userland/capsule_vfs/src/store/fdtable/          open, read, write, lookup, rename, copy, rmdir, chmod
  userland/capsule_vfs/src/store/fdtable/zeroize.rs the volatile erase
  userland/fs_proofs/                              the machine-checked proofs of this source
```

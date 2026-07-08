# VFS Routing and Path Security

Above the filesystems is a dispatcher that decides which one serves a path, and in front of every
path is validation that stops traversal out of the tree. This page documents the VFS routing and the
path defenses. The code is under `src/fs/vfs/`, `src/fs/fd/`, and `src/fs/path/`.

## Routing

The VFS layer (`src/fs/vfs/vfs_core.rs`) is a thin in-kernel dispatcher that routes a path to a
backing filesystem by prefix. `read_file` is representative:

```
  read_file(path):
      if data_rel(path):  blockfs_volume::read_all(path)   // "/data" -> on-disk blockfs
      else:               ramfs::read_file(path)           // everything else -> in-kernel RAMFS
```

And the file-descriptor layer (`src/fs/fd/`) routes the `/ram` tree to the capsule client:

```
  open(path):
      normalize + validate the path
      if is_capsule_path(path):  capsule_client::open(path, flags)   // "/ram" -> ramfs_capsule (IPC)
      else:                      in-kernel RAMFS open
```

So there are three destinations: `/ram` goes over IPC to the [ramfs capsule](vfs-capsule.md), `/data`
goes to the on-disk [blockfs](#the-on-disk-store) store, and everything else is the in-kernel
[RAMFS](ramfs.md). The mount table is a small vector of mount points. The dispatcher itself does no
storage; it decides who does.

## The on-disk store

The `/data` prefix is served by a real on-disk block filesystem, `blockfs`
(`src/fs/blockfs/`): a superblock with a generation, geometry, and a BLAKE3 digest for integrity, a
directory format, and block allocation, optionally encrypted per block through `cryptoblock`
(a 512-byte sector split into a 12-byte nonce, a 16-byte tag, and 484 bytes of ciphertext). It is
honest to record that this on-disk path is dormant in the normal microkernel boot: nothing mounts
`/data` unless an application explicitly reads it, so blockfs and its cryptoblock encryption are built
and correct but off the hot path, which is RAM-resident. The `cryptofs` ephemeral encrypted-file
module is initialized at startup but likewise not on the standard I/O path. The live filesystem is
RAM; the on-disk store is the persistent backend for the code that asks for it.

## Path validation

Every path is validated before use (`src/fs/path/validate.rs`). `validate_path` rejects the malformed,
and `validate_path_secure` adds the traversal check:

```
  validate_path(path):
      reject empty, len > MAX_PATH_LEN (4096), any NUL byte,
             or a component longer than MAX_COMPONENT_LEN (255)

  validate_path_secure(path):
      validate_path(path)
      n = normalize_path(path)
      if n starts with "../" or n == "..":  TraversalAttempt
```

`normalize_path` collapses `.` and `//` and resolves `..` without ever escaping the root, and
`join_secure` refuses an absolute child and rejects a join whose normalized result does not stay under
the parent. Together these stop the classic path-traversal escapes: a `..` that would climb above the
root, a `//` or `.` obfuscation, or an absolute path smuggled in as a relative one. A null byte, which
could truncate a path at a lower layer, is rejected outright.

## Verification

The path canonicalization and the capsule filesystem's access rules are not just asserted, they are
tested against the real source. The `fs_proofs` crate (`userland/fs_proofs/`) includes the actual
capsule filesystem handlers via `#[path]` and runs them: normalization removes `.` and `//` and
resolves `..`, the read-only guard on the signed-capsule directory cannot be bypassed by a trailing
slash or a `..` round-trip, the wire protocol decodes hostile input without panicking, and a fuzz
suite drives millions of inputs. It also includes Kani harnesses for machine-checked proofs. Because
it compiles the production handler source rather than a copy, the proofs are about the code that runs.

## Source

```
  src/fs/vfs/vfs_core.rs        the prefix router (/data, else RAMFS)
  src/fs/fd/                     the fd table and the /ram capsule routing
  src/fs/path/validate.rs        validate_path, validate_path_secure
  src/fs/path/normalize.rs, join.rs   normalize and join_secure
  src/fs/blockfs/, cryptoblock/  the dormant on-disk store
  userland/fs_proofs/            the verification crate
```

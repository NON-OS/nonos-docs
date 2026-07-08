# The Filesystem Capsule

The modern filesystem path in NØNOS is not in the kernel. A client capsule that opens a file under
`/ram` reaches a filesystem capsule over IPC, and that capsule owns the store, the path rules, and the
per-caller access. This page documents the IPC-routed filesystem. The code is under
`src/fs/ramfs_capsule/`, `src/fs/vfs_capsule/`, and `userland/capsule_vfs/`.

## Two services

There are two filesystem capsules, spawned at boot as signed capsules:

```
  ramfs_capsule   service "ramfs",    ports 4096 / 4097,  caps IPC + Memory + Crypto
  vfs_capsule     service "vfs_pool", ports 4104 / 4105,  caps IPC + Memory
```

`ramfs_capsule` serves the `/ram` tree directly and is the lower-level store; `vfs_capsule` is the
higher-level filesystem service application capsules (a terminal, a file manager) talk to. Both are
reached the same way, through the kernel's [IPC](../ipc/README.md) with a named service endpoint and a
reply inbox, and both are ordinary ring-3 capsules the kernel spawns and can tear down.

## The wire protocol

A filesystem operation is an IPC request encoded to a compact wire form. The operations are a small
set (`ramfs_capsule/protocol/types.rs`):

```
  OP_OPEN = 1   OP_CLOSE = 2   OP_READ = 3   OP_WRITE = 4   OP_TRUNCATE = 5
```

The client side (`ramfs_capsule/client/`) encodes the request, an open carries the path as a
UTF-8 C-string plus flags, a read carries a handle, offset, and length, sends it to the service
endpoint, and reads the reply from its inbox. An open returns a remote handle (a capsule-side file
descriptor plus a generation); a read returns the bytes; a write returns a status. The kernel does not
interpret any of this; it routes the message and enforces that the caller holds the capability the
service requires, and the filesystem semantics live entirely in the capsule.

## Caller attestation

Because the store is a shared service, it must know who is asking, and it cannot trust the payload to
say so. The sender identity comes from the kernel, which stamps `proc.<pid>` into every
[IPC envelope](../ipc/envelope.md), not from a field the client fills in. The filesystem capsule reads
the attested caller from the message and applies its access rules against that, so a client cannot
impersonate another to reach its files. The `fs_proofs` suite tests exactly this: a caller cannot forge
another's identity in a request.

## The read-only capsule tree

One access rule is worth calling out: the `/capsules` directory, which holds the signed capsule images,
is read-only. The guard (`userland/capsule_vfs/.../path/is_read_only.rs`) treats `/capsules` and
anything under it as non-writable:

```
  is_read_only(path):  path == "/capsules" or path starts with "/capsules/"
```

This keeps a capsule from rewriting the signed images the system spawns from. The guard is enforced on
the normalized path, and `fs_proofs` proves it cannot be bypassed by a trailing slash (`/capsules//x`)
or a traversal round-trip (`/capsules/../capsules/x`), because the normalization runs first and both
normalize back under `/capsules`. The verified spawn chain still checks every image's
[signatures](../../security/capsules-and-trust.md) at load, so the read-only tree is defense in depth,
not the only guard.

## Source

```
  src/fs/ramfs_capsule/route.rs, protocol/, client/   the /ram IPC filesystem
  src/fs/ramfs_capsule/spawn.rs, vfs_capsule/spawn.rs  the capsule spawns
  userland/capsule_vfs/src/                             the store, server, path handlers
  userland/fs_proofs/                                   the proofs (attestation, /capsules guard)
```

# capsule_ramfs

`capsule_ramfs` is the RAM-resident filesystem service behind the `/ram` tree. It is an IPC request/reply
capsule that owns a map of files entirely in its own heap, and what sets it apart from a plain in-memory
store is that every file is held encrypted at rest: a file's bytes never sit in the capsule as plaintext
except transiently, inside the single decrypted buffer that lives only for the duration of one read or
write. It is a pure userland service capsule with no hardware authority; its capability envelope is IPC,
Memory, and Crypto and nothing else (`userland/capsule_ramfs/Capsule.mk:16`). The source is
`userland/capsule_ramfs/`, mirrored into the kernel at `src/fs/ramfs_capsule/`
(`userland/capsule_ramfs/Capsule.mk:18`).

## Contents

- [Overview and role](#overview-and-role)
- [Identity](#identity)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [The wire protocol](#the-wire-protocol)
- [Operation reference](#operation-reference)
- [The store and the crypto-at-rest model](#the-store-and-the-crypto-at-rest-model)
- [Handles and ownership](#handles-and-ownership)
- [Relationship to the vfs](#relationship-to-the-vfs)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview and role

The capsule is a single-threaded request/reply server. `_start` initializes the heap and calls
`server::run`, which never returns (`src/main.rs:30`, `src/server/runner.rs:28`). The server holds two
pieces of state for its whole life: a `Store` that maps a path string to an encrypted file, and a
`HandleTable` that maps a 64-bit handle to a path plus the pid that opened it (`src/server/runner.rs:30`).
Everything the capsule does is one of five operations against those two structures.

The role is narrow on purpose. This is not the general filesystem the desktop talks to; that is the
[vfs pool](../vfs/README.md). This capsule is the store the kernel routes to when a path lives under `/ram`, and it
exists to give the system a fast, ephemeral, per-file-encrypted scratch tree that never touches a block
device. It holds the Crypto capability because it does its own encryption at rest; it holds IPC because it
is a service; it holds Memory because its files are its heap. It holds nothing else, and in particular no
FileSystem, Driver, Mmio, Dma, Pio, or Irq capability, so it cannot reach a storage device or any hardware
at all.

## Identity

Everything the kernel and the service registry need to name and reach the capsule comes from its
`Capsule.mk` and the kernel-side spawn record that embeds and verifies it.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `ramfs` | `Capsule.mk:7` |
| Service handle | `ramfs` | `Capsule.mk:8` |
| Namespace | `systems.nonos.ramfs` | `Capsule.mk:13` |
| Service endpoint | `service:4096:ramfs` | `Capsule.mk:14`, `src/fs/ramfs_capsule/spawn.rs:32` |
| Reply endpoint | `reply:4097:endpoint.4294967297` | `Capsule.mk:15`, `spawn.rs:33` |
| Reply inbox name | `endpoint.4294967297` | `src/fs/ramfs_capsule/client/transport.rs:25` |
| Capability mask | `0x38` | `Capsule.mk:16`, `spawn.rs:50` |
| Binary name | `ramfs` | `Capsule.mk:11` |
| Kernel mirror | `src/fs/ramfs_capsule` | `Capsule.mk:18` |

Two numbers that look like they should match do not, and it is worth being precise. The service port is
4096 and the reply port is 4097 (`spawn.rs:32`, `spawn.rs:33`). The reply endpoint's inbox name is the
decimal string `endpoint.4294967297`, and `4294967297` is `0x1_0000_0001`, the same value the capsule's
own protocol names `KERNEL_REPLY_ENDPOINT` (`src/protocol/types.rs:26`). So the capsule receives requests
on its service inbox, and every reply is sent to the single well-known endpoint `0x1_0000_0001` where the
kernel-side client is listening.

The mask `0x38` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x08  IPC       bit()  8    types.rs:59
  0x10  Memory    bit() 16    types.rs:60
  0x20  Crypto    bit() 32    types.rs:61
  ----
  0x38  = 8 + 16 + 32
```

The kernel spawn path requests exactly those three capabilities and no others, assembled from the same
`Capability::bit()` values (`spawn.rs:50`): `Capability::IPC.bit() | Capability::Memory.bit() |
Capability::Crypto.bit()`. There is no FileSystem bit (64), no hardware or driver bit, and no
RegisterService or Debug bit. The whole security posture below rests on that list.

## Architecture and lifecycle

The crate is `no_std`/`no_main` and has four top-level modules: `handles`, `protocol`, `server`, and
`store` (`src/main.rs:22`).

- `protocol` is the wire codec: the five opcodes and the open flags (`src/protocol/types.rs`), the
  request decoder (`src/protocol/decode.rs`), the response encoder (`src/protocol/encode.rs`), and the
  errno set (`src/protocol/errno.rs`).
- `server` is the loop and the dispatch: `runner.rs` owns the receive/dispatch/reply cycle,
  `dispatch.rs` maps an opcode to a handler, and `handlers/` holds one file per operation.
- `store` is the encrypted file map: `types.rs` defines the `File` and `Store`, `state.rs` holds
  `new`/`contains`/`ensure`, `read.rs`/`write.rs`/`truncate.rs` are the three data paths, and `crypto/`
  holds the AEAD wrappers.
- `handles` is the pid-owned descriptor table (`src/handles.rs`).

The server loop is the shape every core service capsule shares (`src/server/runner.rs:28`):

```
  run():
      buf     = vec![0; 8192]             // MAX_MSG
      store   = Store::new()              // BTreeMap<String, File>
      handles = HandleTable::new()        // BTreeMap<u64, Handle>, next_id = 1
      loop:
          sender_pid = 0
          n = mk_ipc_recv_from(inbox 0, buf, 8192, 0, &sender_pid)   // kernel stamps sender_pid
          if n <= 0: continue
          req = decode_request(buf[..n])   // drop on a short or malformed header
          resp = dispatch(store, handles, req, sender_pid)
          mk_ipc_send(KERNEL_REPLY_ENDPOINT, resp)
```

The `sender_pid` is stamped by the kernel on receive, not carried in the payload (`src/server/runner.rs:33`),
which is what makes the handle-ownership check below trustworthy. A message that does not even decode is
dropped silently and the loop moves on (`runner.rs:38`); the capsule never panics on bad input.

Lifecycle: the capsule is spawned very early in the boot fleet, before the services that depend on a scratch
store. `spawn_ramfs` calls the shared boot helper as `boot::capsule("RAMFS", "ramfs", ...)`
(`src/userspace/init/spawn_plan/core.rs:17`), which runs `capsule_boot::boot`. That helper verifies the
embedded ELF, id cert, manifest, and attestation trailer through `spawn_verified`
(`src/fs/ramfs_capsule/spawn.rs:36`), and on success prints `[RAMFS] capsule spawned`
(`src/userspace/init/capsule_boot/run.rs:29`); on failure it prints an `[ERROR]` line carrying the mapped
`SpawnError` (`run.rs:32`). The boot ordering is explicit: `spawn_ramfs` runs, then `spawn_core_after_ramfs`
brings up keyring, entropy, crypto, and policy (`src/userspace/init/entry.rs:25`,
`src/userspace/init/spawn_plan/core.rs:22`).

## The wire protocol

Every request is an 8-byte header followed by an operation-specific payload
(`src/protocol/types.rs:28`, `src/protocol/decode.rs:19`):

```
  offset 0  u32  seq        little-endian sequence, echoed in the reply
  offset 4  u16  op         the operation
  offset 6  u16  reserved   must be zero, else the request is dropped
```

`decode_request` rejects any buffer shorter than the header and any request whose reserved field is
non-zero (`decode.rs:20`, `decode.rs:24`); a rejected request is dropped without a reply. Every reply is
the same header-less shape (`src/protocol/encode.rs:21`):

```
  offset 0  u32  seq        echoed from the request
  offset 4  i32  status     >= 0 on success, a negative errno on failure
  offset 8  ...  body       operation-specific, may be empty
```

The `status` field carries two meanings depending on the op. For read it is the byte count and the body is
the bytes; for write it is the byte count and the body is empty; for open it is 0 and the body is the
8-byte handle; for close and truncate it is 0 with an empty body. A negative status is always one of the
five errnos (`src/protocol/errno.rs`):

```
  ENOENT  -2    no such file or handle
  EIO     -5    a crypto seal/open failed (a bad tag or a random-source error)
  EACCES  -13   a handle owned by another pid
  EINVAL  -22   a malformed payload or an unknown op
  EMFILE  -24   the handle table is full
```

An unknown op falls through `dispatch` to `EINVAL` (`src/server/dispatch.rs:38`).

## Operation reference

The five operations are dispatched by opcode in `dispatch` (`src/server/dispatch.rs:32`).

### OP_OPEN = 1

`src/protocol/types.rs:17`. Handler `src/server/handlers/open.rs:27`.

Request payload:

```
  offset 0  u32  flags       CREATE = 0x1, TRUNCATE = 0x2  (types.rs:23, types.rs:24)
  offset 4  u16  path_len
  offset 6  ..   path        path_len bytes of UTF-8
```

The handler needs at least 6 header bytes and then `path_len` more, and it validates the path as UTF-8
(`open.rs:33`, `open.rs:44`, `open.rs:47`). If the file does not exist and `CREATE` is not set it returns
`ENOENT`; if `CREATE` is set it calls `store.ensure`, which fails to `EIO` only if the crypto random source
fails while minting the file's key or nonce (`open.rs:52`, `open.rs:55`). If `TRUNCATE` is set the file is
truncated to zero (`open.rs:59`). On success the handler allocates a handle tagged with the sender pid and
replies `status = 0` with the 8-byte little-endian handle as the body; a full table returns `EMFILE`
(`open.rs:62`).

### OP_READ = 3

`src/protocol/types.rs:19`. Handler `src/server/handlers/read.rs:25`.

Request payload (at least 20 bytes, `read.rs:26`):

```
  offset 0   u64  handle
  offset 8   u64  offset
  offset 16  u32  count
```

The handler resolves the handle to a path scoped to the sender pid (`read.rs:41`): a handle owned by
another pid is `EACCES`, an absent handle is `ENOENT`. It then calls `store.read_at`, which decrypts the
whole file and returns the `[offset .. offset+count]` slice (`read.rs:46`). On success `status` is the byte
count and the body is the bytes; a missing file is `ENOENT` and a crypto failure is `EIO` (`read.rs:47`).
An offset past the end returns an empty read, not an error (`src/store/read.rs:32`).

### OP_WRITE = 4

`src/protocol/types.rs:20`. Handler `src/server/handlers/write.rs:24`.

Request payload (at least 16 bytes, `write.rs:30`):

```
  offset 0   u64  handle
  offset 8   u64  offset
  offset 16  ..   data       the rest of the payload
```

Same pid-scoped handle resolution (`write.rs:42`). `store.write_at` decrypts the file, splices the new
bytes at `offset` (zero-filling any gap past the old end), draws a fresh nonce, and re-encrypts the whole
plaintext (`write.rs:47`, `src/store/write.rs:24`). On success `status` is the number of bytes written and
the body is empty; a missing file is `ENOENT` and a crypto failure is `EIO` (`write.rs:48`). Note that
write auto-creates the file if it is not present, because `write_at` calls `ensure` first
(`src/store/write.rs:30`).

### OP_TRUNCATE = 5

`src/protocol/types.rs:21`. Handler `src/server/handlers/truncate.rs:24`.

Request payload (at least 16 bytes, `truncate.rs:30`):

```
  offset 0  u64  handle
  offset 8  u64  length
```

Same pid-scoped handle resolution (`truncate.rs:41`). `store.truncate` decrypts, resizes the plaintext to
`length` (growing with zeros or shrinking), draws a fresh nonce, and re-encrypts; a resize to zero drops
the ciphertext entirely (`src/store/truncate.rs:24`, `truncate.rs:36`). On success `status` is 0 with an
empty body; a missing file is `ENOENT` and a crypto failure is `EIO` (`truncate.rs:46`).

### OP_CLOSE = 2

`src/protocol/types.rs:18`. Handler `src/server/handlers/close.rs:22`.

Request payload (at least 8 bytes, `close.rs:23`):

```
  offset 0  u64  handle
```

`handles.remove` drops the handle after the same owner check (`close.rs:30`): success is `status = 0`,
a handle owned by another pid is `EACCES`, and an absent handle is `ENOENT`. Close touches the handle
table only; the file's ciphertext stays in the store.

## The store and the crypto-at-rest model

The `Store` is a `BTreeMap<String, File>` keyed by path, and a `File` holds only ciphertext plus the key
and nonce that seal it (`src/store/types.rs:23`):

```
  struct File  { key: [u8; 32], nonce: [u8; 12], ciphertext: Vec<u8> }
  struct Store { files: BTreeMap<String, File> }
  enum StoreError { NotFound, CryptoFailure }
```

There is no plaintext field. A file exists in the store only as ChaCha20-Poly1305 ciphertext, with its own
32-byte key and 12-byte nonce, and the ciphertext includes the 16-byte Poly1305 tag
(`src/store/crypto/constants.rs:17`, `constants.rs:18`, `constants.rs:19`).

The encryption is real and it goes through the kernel. `seal` and `open` call the `nonos_libc` syscalls
`crypto_encrypt` and `crypto_decrypt` with the algorithm id `ALGO_CHACHA20_POLY1305 = 0`
(`src/store/crypto/seal.rs:28`, `src/store/crypto/open.rs:28`, `constants.rs:16`). The key and nonce are
drawn from the kernel's random source through `crypto_random` (`src/store/crypto/fresh_key.rs:24`,
`src/store/crypto/fresh_nonce.rs:24`); any failure of that source returns `StoreError::CryptoFailure`,
which every handler maps to `EIO`. This is a plain AEAD call with no associated data.

The key and nonce discipline is the correctness argument:

- The key is minted once, when the file is created, in `ensure` (`src/store/state.rs:37`). It does not
  rotate over the file's life.
- The nonce is minted at creation and then redrawn on every write and every truncate, before the
  re-encryption (`src/store/write.rs:45`, `src/store/truncate.rs:35`). A fresh nonce on every seal is what
  keeps a nonce from ever being reused under the same key, which is the requirement the AEAD needs to stay
  secure.
- A failed authentication on decrypt surfaces as `CryptoFailure` mapped to `EIO`
  (`src/store/read.rs:30`, `read.rs`), so the store never hands back unauthenticated bytes.

The data paths all share the decrypt/operate/re-encrypt shape. A read decrypts the whole file into a
transient `plain` buffer and slices it, without re-encrypting (`src/store/read.rs:24`). A write decrypts,
splices, resizes if the write extends past the end, then seals under a fresh nonce (`src/store/write.rs:24`).
A truncate decrypts, resizes, and seals under a fresh nonce, or drops the ciphertext if the new length is
zero (`src/store/truncate.rs:24`). So the on-heap representation is always ciphertext except for that one
transient `plain` buffer that lives for the duration of a single operation.

## Handles and ownership

An open allocates a 64-bit handle in the `HandleTable` (`src/handles.rs:37`) tagged with the caller's pid,
and read, write, truncate, and close resolve a handle only for its owner:

```
  insert(path, owner_pid):    reject with None if table full (MAX_HANDLES = 1024); else assign next_id
  path_for(id, sender_pid):   NotFound if absent; Denied if owner_pid != sender_pid (unless sender_pid == 0)
  remove(id, sender_pid):     same owner check
```

The table caps at 1024 handles (`src/handles.rs:20`, `handles.rs:38`), and `insert` returning `None` is
what surfaces as `EMFILE`. The `next_id` starts at 1 and only ever increments with `wrapping_add`, so ids
are not reused across a live table (`handles.rs:34`, `handles.rs:42`). `path_for` and `remove` both refuse
a handle whose stored `owner_pid` does not match the attested `sender_pid`, with one exception: a
`sender_pid` of 0 bypasses the check (`handles.rs:49`, `handles.rs:58`), which is the kernel-side path,
where the kernel client speaks to the store on behalf of the fd layer. So one ordinary capsule cannot read
or write another capsule's open file through a guessed handle, because the identity comes from the
kernel-attested `sender_pid`, never from a payload field.

## Relationship to the vfs

`/ram` is not served by the [vfs pool](../vfs/README.md); it is served by this capsule, and the routing lives in the
kernel's file-descriptor layer, not in the vfs capsule. The kernel decides a path belongs to this store
with `is_capsule_path`, which is true for exactly `/ram` and any path under `/ram/`
(`src/fs/ramfs_capsule/route.rs:17`). When the fd layer sees such a path it calls the kernel-side ramfs
client (`src/fs/fd/table/open.rs:25`, `src/fs/fd/syscalls/read.rs:20`, `.../write.rs:25`,
`src/fs/fd/table/truncate.rs:20`) rather than the in-kernel filesystem. That client speaks the exact wire
protocol above and sends every reply to the `endpoint.4294967297` inbox
(`src/fs/ramfs_capsule/client/transport.rs:25`). The kernel client sends under the name `kernel.ramfs` and
runs with an effective `sender_pid` of 0, which is why the store lets pid 0 bypass the handle-owner check.

This is worth stating plainly because it is easy to assume the vfs owns `/ram`. It does not. The vfs pool
owns its own store and its own path tree; the `/ram` subtree is a separate capsule reached through a
separate kernel client. The two are peers, not layers.

## Security analysis

The capsule is zero-state and RAM-resident by construction: its entire contents are a `BTreeMap` in its
own heap, there is no persistence path, and a restart starts from an empty store. Its authority is exactly
IPC, Memory, and Crypto (`Capsule.mk:16`, `spawn.rs:50`), so it cannot reach a block device, a socket, a
device register, or a DMA channel; it has no FileSystem, Network, Driver, Mmio, Dma, Pio, Irq, or Debug
capability at all. It cannot register a service name beyond the one its verified manifest declares, because
it has no RegisterService bit.

- Encryption at rest: a file is ChaCha20-Poly1305 ciphertext in the heap, with its own key and nonce
  (`src/store/types.rs:23`). Plaintext exists only in the transient `plain` buffer during a single read,
  write, or truncate.
- Nonce discipline: a fresh nonce on every seal (`src/store/write.rs:45`, `src/store/truncate.rs:35`)
  means no nonce is ever reused under a key, the correctness condition for the AEAD.
- Authentication: a bad tag is `CryptoFailure` mapped to `EIO` (`src/store/read.rs:30`), never a return of
  unauthenticated bytes.
- Handle ownership: a descriptor is usable only by the pid that opened it, checked per access against the
  kernel-attested `sender_pid` (`src/handles.rs:49`).
- Isolation: a file lives only in this capsule's heap as ciphertext, and another capsule reaches it only by
  sending a request that the owner check scopes to the caller pid. Cross-capsule isolation is the kernel's,
  the same boundary every capsule sits behind.

Honest gaps, stated plainly:

- The transient `plain` buffer during a read, write, or truncate is a `Vec` and is not explicitly zeroized
  after use (Rust's `Vec` drop does not zero its backing), so a file's plaintext can linger in the heap
  until that allocation is reused. This is the one place the encryption-at-rest posture is a heap-lifetime
  property rather than an absolute one.
- There is no per-file size cap and no total-store cap: a file can grow unbounded until the heap is
  exhausted (`src/store/write.rs:41`).
- The pid-0 bypass in the handle check is a real trust assumption: it exists so the kernel client can drive
  the store, and it is safe only because a `sender_pid` of 0 is stamped by the kernel, not chosen by a
  caller (`src/handles.rs:49`).
- The key does not rotate over a file's life; only the nonce does (`src/store/state.rs:37`,
  `src/store/write.rs:45`).
- There is no persistence: the store is empty after a restart by design.

## How to contribute

The source lives at `userland/capsule_ramfs/`. To change the protocol, edit `src/protocol/`; to change the
server behaviour, edit `src/server/`; to change the store or the crypto-at-rest path, edit `src/store/`
and `src/store/crypto/`; the handle table is `src/handles.rs`.

To add an operation:

1. Add the opcode constant to `src/protocol/types.rs:17` and re-export it from `src/protocol/mod.rs:25`.
2. Write the handler as one file under `src/server/handlers/`, following the existing pattern: validate the
   payload length, decode the fields with the `read_*_le` helpers, resolve the handle with
   `handles.path_for` when the op takes one, call into the store, and map every `StoreError` to an errno.
   Re-export it from `src/server/handlers/mod.rs:23`.
3. Wire the opcode into the match in `src/server/dispatch.rs:32`.
4. If the op needs a new store method, add it as one file under `src/store/` alongside `read.rs`,
   `write.rs`, and `truncate.rs`, and keep the decrypt/operate/re-encrypt discipline: draw a fresh nonce
   before every seal.

To build and sign the capsule, use the per-slug make targets that `nonos-mk/capsule.mk` generates from the
`ramfs` slug (`nonos-mk/capsule.mk:158`, `:182`, `:261`, `:263`, included through
`userland/capsule_ramfs/Capsule.mk:20`):

```
  make nonos-mk-ramfs               build the capsule ELF
  make nonos-mk-ramfs-sign          produce the id cert, manifest, and attestation trailer
  make nonos-mk-ramfs-verify        verify the signed manifest against the trust anchor
  make nonos-mk-check-ramfs-keys    check the per-capsule signing keys exist
```

For a running image, `make nonos-mk-ramfs-prod` builds a kernel profile with the `microkernel-ramfs`
feature and the ramfs artifacts staged (`Makefile:905`), and `make nonos-mk-boot-ramfs` runs the
round-trip boot harness (`Makefile:1381`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns an errno, never a panic, and a malformed request is dropped
in the loop); modular files, one unit per file, with `mod.rs` used only for re-exports; and the AGPL header
at the top of every source file, matching the header on every existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints `[RAMFS] capsule
spawned` (tag `RAMFS`, message `capsule spawned`) from the boot log (`src/userspace/init/capsule_boot/run.rs:29`,
`src/sys/boot_log/output.rs:33`, where `ok` writes `[<tag>] <msg>` to the serial port). That line is
mirrored to the framebuffer console under `NONOS_FBCONSOLE=1` on a machine with no serial port. An absent
line means the capsule never started, usually a signature, manifest, or capability failure; the error path
prints an `[ERROR]` line carrying the mapped `SpawnError` instead (`run.rs:32`,
`src/sys/boot_log/output.rs:49`). If the marker is present the service is registered and `/ram` access
works; if it is absent, the ELF failed verification or the `0x38` mask fell outside policy, and no `/ram`
path resolves.

Failure signatures at request time are the five errnos:

- `EINVAL` (-22): a payload shorter than the op requires, a non-UTF-8 open path, or an unknown op
  (`src/server/handlers/open.rs:47`, `src/server/dispatch.rs:38`). A request whose reserved field is
  non-zero does not even get a reply; it is dropped in the decoder (`src/protocol/decode.rs:24`).
- `ENOENT` (-2): an open without `CREATE` on a missing file (`open.rs:53`), or a handle that does not exist.
- `EACCES` (-13): a handle owned by another pid, the owner check refusing a guessed handle
  (`src/handles.rs:49`).
- `EMFILE` (-24): the handle table is full at 1024 entries (`src/handles.rs:38`).
- `EIO` (-5): a crypto seal or open failed. On a healthy store this is the tell that a file's authentication
  tag did not verify, which means memory corruption rather than an ordinary miss, or that the kernel random
  source failed while minting a key or nonce (`src/store/read.rs:30`, `src/store/crypto/fresh_key.rs:24`).

The kernel-side spawn embeds a distinct debug tag, `[RAMFS-DEBUG] load_elf_executable error:`, that
surfaces an ELF-load failure during verification (`src/fs/ramfs_capsule/spawn.rs:51`), which separates an
ELF problem from a signature or capability problem.

## Source map

```
  userland/capsule_ramfs/src/main.rs                _start -> heap_init -> server::run
  userland/capsule_ramfs/src/server/runner.rs       the recv/dispatch/reply loop
  userland/capsule_ramfs/src/server/dispatch.rs     op -> handler
  userland/capsule_ramfs/src/server/handlers/       open, read, write, truncate, close
  userland/capsule_ramfs/src/protocol/types.rs      opcodes, open flags, header, KERNEL_REPLY_ENDPOINT
  userland/capsule_ramfs/src/protocol/decode.rs     request decode + read_*_le helpers
  userland/capsule_ramfs/src/protocol/encode.rs     response encode
  userland/capsule_ramfs/src/protocol/errno.rs      ENOENT EIO EACCES EINVAL EMFILE
  userland/capsule_ramfs/src/store/types.rs         File (key, nonce, ciphertext), Store, StoreError
  userland/capsule_ramfs/src/store/state.rs         new, contains, ensure (mints key + nonce)
  userland/capsule_ramfs/src/store/{read,write,truncate}.rs   the decrypt/operate/re-encrypt paths
  userland/capsule_ramfs/src/store/crypto/          constants, fresh_key, fresh_nonce, seal, open
  userland/capsule_ramfs/src/handles.rs             the pid-owned handle table
  Capsule.mk                                        slug, handle, ports, capability mask, kernel mirror
  src/fs/ramfs_capsule/spawn.rs                     the kernel-side embed and verified spawn
  src/fs/ramfs_capsule/route.rs                     is_capsule_path: /ram and /ram/*
  src/fs/ramfs_capsule/client/                      the kernel client the fd layer calls
  src/fs/fd/                                         the fd-layer routing of /ram to this capsule
  src/userspace/init/spawn_plan/core.rs             the boot-fleet spawn entry
  nonos-mk/capsule.mk                               the generated nonos-mk-ramfs[-sign|-verify] targets
```

Every reference above is verified against those trees.
</content>
</invoke>

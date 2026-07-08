# capsule_ramfs

`capsule_ramfs` is the IPC-routed RAM filesystem behind the `/ram` tree. What distinguishes it from a
plain in-memory store is that every file is encrypted at rest with its own key: a file's bytes never sit
in the capsule's heap as plaintext except transiently during a read or write. It is the lower-level store
under the [vfs pool](vfs.md)'s `/ram` routing. Service `ramfs` on port 4096, reply endpoint
`0x1_0000_0001`, capability mask `0x38`. The source is `userland/capsule_ramfs/`.

## Contents

- [The server loop](#the-server-loop)
- [The wire protocol](#the-wire-protocol)
- [The store model](#the-store-model)
- [Per-file encryption](#per-file-encryption)
- [Read and write](#read-and-write)
- [Handles and ownership](#handles-and-ownership)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop

`main.rs:31` initializes the heap and calls `server::run` (`src/server/runner.rs:28`) with an 8 KiB
buffer:

```
  run():
      store   = Store::new()          // BTreeMap<String, File>
      handles = HandleTable::new()    // BTreeMap<u64, Handle{path, owner_pid}>, next_id = 1
      loop:
          n = mk_ipc_recv_from(inbox 0, buf, &sender_pid)
          req = decode_request(buf[..n])          // 8-byte header: u32 seq, u16 op, u16 reserved
          resp = dispatch(store, handles, req, sender_pid)
          mk_ipc_send(KERNEL_REPLY_ENDPOINT, resp)
```

The header is a compact eight bytes (a sequence and an op, reserved must be zero); the response is the
sequence, an `i32` status, and an optional body.

## The wire protocol

Five operations (`src/protocol/types.rs:17`):

```
  1  OPEN (flags: CREATE=0x1, TRUNCATE=0x2, then path)
  2  CLOSE (handle)
  3  READ  (handle, u64 offset, u32 count)
  4  WRITE (handle, u64 offset, bytes)
  5  TRUNCATE (handle, u64 size)
```

`dispatch` (`src/server/dispatch.rs`) routes each; an unknown op is `EINVAL`. The errnos are the POSIX
set (`ENOENT`, `EIO`, `EACCES`, `EINVAL`, `EMFILE`).

## The store model

The `Store` (`src/store/types.rs:29`) is a `BTreeMap<String, File>` keyed by path, and a `File` holds
only ciphertext plus its key and nonce:

```
  struct File  { key: [u8; 32], nonce: [u8; 12], ciphertext: Vec<u8> }
  enum StoreError { NotFound, CryptoFailure }
```

There is no plaintext field: a file exists in the store only as ChaCha20-Poly1305 ciphertext with its own
32-byte key (generated at open) and a 12-byte nonce (refreshed on every write). The ciphertext includes
the 16-byte Poly1305 authentication tag.

## Per-file encryption

Each file has a unique key and nonce (`src/store/crypto.rs`, `KEY_LEN = 32`, `NONCE_LEN = 12`). The key is
generated once when the file is created and does not rotate per write; the nonce is redrawn on every
write, which is what keeps a nonce from ever being reused under the same key, the essential requirement
for the AEAD's security. Encryption and decryption go through the kernel's AEAD (`crypto_encrypt_aad` /
`crypto_decrypt_aad`), and a failed authentication tag surfaces as `StoreError::CryptoFailure`, mapped to
`EIO`, so the store never returns unauthenticated plaintext.

## Read and write

Read and write decrypt the whole file, operate, and (for write) re-encrypt with a fresh nonce:

```
  read_at(path, offset, count):
      plain = decrypt(file.key, file.nonce, file.ciphertext)   // EIO on tag failure
      return plain[offset .. offset+count]                      // never re-encrypts

  write_at(path, offset, data):
      plain = decrypt(file.key, file.nonce, file.ciphertext)
      grow plain to offset+len (zero-fill any gap); splice data at offset
      nonce' = fresh_nonce()
      file.ciphertext = encrypt(file.key, nonce', plain)        // fresh nonce every write
      file.nonce = nonce'
```

A read decrypts and slices without re-encrypting; a write decrypts, splices the new bytes (zero-filling
any gap past the old end), draws a fresh nonce, and re-encrypts the whole plaintext. So the on-heap
representation is always the ciphertext except for the transient decrypted `plain` buffer during the
operation.

## Handles and ownership

An open allocates a 64-bit handle in the `HandleTable` (`src/handles.rs:27`) tagged with the caller's
pid, and read, write, and close resolve a handle only for its owner:

```
  insert(path, owner_pid):  reject if table full (MAX_HANDLES = 1024); else assign next_id
  path_for(id, sender_pid): NotFound if absent; Denied if owner_pid != sender_pid (unless sender_pid == 0)
  remove(id, sender_pid):   same owner check
```

So one capsule cannot read or write another capsule's open file through a guessed handle, because the
handle table checks the attested sender pid on every access (the kernel-side mirror, pid 0, bypasses the
check). The identity comes from the kernel-attested `sender_pid`, not a payload field.

## Security analysis

- **Encryption at rest**: a file is ChaCha20-Poly1305 ciphertext in the heap; plaintext exists only in a
  transient decrypted buffer during a read or write.
- **Nonce discipline**: a fresh nonce on every write means no nonce reuse under a key, which is the
  correctness condition for the AEAD.
- **Authentication**: a bad tag is `EIO`, never a return of unauthenticated bytes.
- **Handle ownership**: a descriptor is usable only by its owner pid, checked per access.

## Honest gaps

Stated plainly: the transient decrypted `plain` buffer during a read or write is a `Vec` and is not
explicitly zeroized after use (Rust's `Vec` drop does not zero its backing), so a file's plaintext can
linger in the heap until that allocation is reused, this is the one place the encryption-at-rest posture
has a gap. There is no per-file size cap, so a file can grow unbounded; the key does not rotate across the
file's life (only the nonce does); and there is no persistence across a restart. The `/ram` routing and
the kernel-side view are on the [storage](../../subsystems/storage/vfs-capsule.md) page.

## Source map

```
  userland/capsule_ramfs/src/server/runner.rs      the loop
  userland/capsule_ramfs/src/server/dispatch.rs     op -> handler
  userland/capsule_ramfs/src/server/handlers/       open, read, write, close, truncate
  userland/capsule_ramfs/src/store/types.rs          File (key, nonce, ciphertext), Store, StoreError
  userland/capsule_ramfs/src/store/crypto.rs         KEY_LEN, NONCE_LEN, the AEAD seal/open
  userland/capsule_ramfs/src/store/read_at.rs, write_at.rs   the decrypt/splice/re-encrypt paths
  userland/capsule_ramfs/src/handles.rs              the pid-owned handle table
```

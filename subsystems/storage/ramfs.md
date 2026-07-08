# RAMFS

The default filesystem in NØNOS is in RAM. A microkernel that boots RAM-resident and leans on the
[ZeroState](../memory/zeroization.md) posture keeps its live filesystem in memory, and RAMFS is that
filesystem: an in-kernel tree of files whose contents are zeroed the moment they are freed. This page
documents it. The code is under `src/fs/ramfs/`.

## The file

A RAMFS file is a `NonosFile` (`src/fs/ramfs/types.rs`), a name, a byte vector, and POSIX metadata:

```
  struct NonosFile {
      name: String, data: Vec<u8>, size: usize,
      created, modified: u64,
      encrypted, quantum_protected: bool,
      mode, uid, gid: u32,
  }
```

The filesystem is a tree of these behind a global instance, and it supports the expected operations,
`create_file`, `read_file`, `write_file`, `create_dir`, `mkdir_all`, `delete_file`, `list_dir`
(`ramfs/mod.rs`). It is the default: every path that is not routed to a capsule or to the on-disk
`/data` store is served here in the kernel.

## Zero on free

The property that ties RAMFS to the kernel's RAM-residency claim is that a file's bytes are zeroed
when the file is dropped. `secure_zeroize` (`types.rs`) overwrites the data with volatile writes and a
compiler fence, `secure_clear` calls it and clears the vector, and `Drop` calls `secure_clear`:

```
  secure_zeroize(data):
      for byte in data: write_volatile(byte, 0)
      compiler_fence(SeqCst)

  impl Drop for NonosFile:
      fn drop(&mut self):  secure_clear(self)   // zeroize + clear + size = 0
```

The volatile write is what keeps the compiler from optimizing the wipe away as a dead store, and the
fence orders it, so a freed file's contents do not linger in the heap to be read back by a later
allocation. This is the per-file half of the [zeroization](../memory/zeroization.md) story: RAMFS
does not depend on the whole-system wipe to avoid leaving file data in RAM, because each file zeroes
itself on the way out. It complements the [heap](../memory/heap.md)'s own zero-on-free.

## Where it sits

RAMFS is reached two ways: directly, as the in-kernel default for kernel-side paths, and through the
[VFS dispatcher](vfs-and-paths.md), which routes most paths to it. The IPC-facing `/ram` tree is a
separate capsule (`ramfs_capsule`), documented on the [VFS capsule](vfs-capsule.md) page; that capsule
serves the modern capsule-client filesystem while this in-kernel RAMFS backs the kernel's own paths.

## Source

```
  src/fs/ramfs/types.rs             NonosFile, secure_zeroize, the Drop wipe
  src/fs/ramfs/mod.rs                the file and directory operations
  src/fs/ramfs/filesystem/global.rs  the global RAMFS instance
```

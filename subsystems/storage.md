# Storage

This page describes storage-related userland capsules: block drivers, USB mass
storage command helpers, RAMFS, and VFS. Read
[Userland Model](../userland/README.md) and [Syscall ABI Reference](../abi/syscalls.md)
first.

---

## 1. Spawn order

Virtio block is part of the virtio I/O driver group. That group spawns RNG and
block drivers when enabled (`src/userspace/init/spawn_plan/drivers_virtio_io.rs:17`).
The storage driver group spawns AHCI, HDA, and NVMe when their features are
enabled (`src/userspace/init/spawn_plan/drivers_storage.rs:17`). USB mass
storage is part of the USB driver group after xHCI and USB HID
(`src/userspace/init/spawn_plan/drivers_usb.rs:17`).

VFS starts after drivers and before network in `run_init`
(`src/userspace/init/entry.rs:26`). RAMFS is spawned before core services
(`src/userspace/init/entry.rs:23`).

```
  RAMFS
    |
  core services
    |
  block and USB drivers
    |
  VFS
```

## 2. Block driver capsules

| Capsule | Entry behavior | Protocol |
|---------|----------------|----------|
| virtio-blk | Heap init, retry setup with yields, run server with mutable driver (`userland/capsule_driver_virtio_blk/src/main.rs:29`) | healthcheck, capacity, read blocks, write blocks, flush (`userland/capsule_driver_virtio_blk/src/protocol/ops.rs:16`) |
| AHCI | Heap init, setup or exit with mapped error, run server (`userland/capsule_driver_ahci/src/main.rs:36`) | healthcheck, controller info, port list (`userland/capsule_driver_ahci/src/protocol/ops.rs:17`) |
| NVMe | Heap init, setup or exit with mapped error, run server (`userland/capsule_driver_nvme/src/main.rs:38`) | healthcheck, controller info, identify controller, identify namespace, SMART health (`userland/capsule_driver_nvme/src/protocol/ops.rs:17`) |
| USB MSC | Heap init, run server (`userland/capsule_driver_usb_msc/src/main.rs:31`) | probe config, build inquiry, build read capacity 10, build read 10, build write 10, accept CSW, get state (`userland/capsule_driver_usb_msc/src/protocol/ops.rs:17`) |

## 3. RAMFS

RAMFS is a no_std, no_main capsule with handles, protocol, server, and store
modules (`userland/capsule_ramfs/src/main.rs:17`). It initializes heap and
enters `server::run` (`userland/capsule_ramfs/src/main.rs:29`).

The RAMFS protocol exports request decode and response encode helpers, errno
values, request type, kernel reply endpoint, open flags, and five ops: open,
close, read, write, and truncate (`userland/capsule_ramfs/src/protocol/mod.rs:22`).
The op values and open flags are defined in protocol types
(`userland/capsule_ramfs/src/protocol/types.rs:17`).

## 4. VFS

VFS is a no_std, no_main capsule with protocol, server, and store modules
(`userland/capsule_vfs/src/main.rs:17`). It initializes heap and enters
`server::run` (`userland/capsule_vfs/src/main.rs:28`).

The VFS protocol exports request decode and response encode helpers, errno
values, request type, kernel reply endpoint, max path, data, and list sizes, and
ops for open, close, read, write, stat, list, healthcheck, mkdir, unlink, and
rename (`userland/capsule_vfs/src/protocol/mod.rs:22`). The type module defines
magic value `0x4E4F_5646`, version `1`, flags for create, truncate, and append,
and the 20-byte header length (`userland/capsule_vfs/src/protocol/types.rs:17`).

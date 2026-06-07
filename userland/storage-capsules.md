# Storage Capsules

This page documents the storage-facing capsule set: RAMFS, VFS, virtio block,
AHCI, NVMe, and USB mass storage. Read [Core Service Capsules](core-capsules.md),
[Drivers](drivers.md), and [Storage](../subsystems/storage.md) first.

The storage path is split deliberately. File state is owned by RAMFS and VFS.
Block device control is owned by driver capsules. USB mass storage is a command
builder and state tracker that sits behind xHCI and USB descriptors.

---

## 1. Storage Stack Shape

RAMFS starts before core services, VFS starts after drivers and before network,
virtio block starts in the virtio I/O driver group, AHCI and NVMe start in the
storage driver group, and USB MSC starts in the USB group after xHCI and USB HID
(`src/userspace/init/entry.rs:23`,
`src/userspace/init/entry.rs:26`,
`src/userspace/init/spawn_plan/drivers_virtio_io.rs:17`,
`src/userspace/init/spawn_plan/drivers_storage.rs:17`,
`src/userspace/init/spawn_plan/drivers_usb.rs:17`).

```
+--------------------------+
| ramfs                    |
+------------+-------------+
             |
+------------+-------------+
| core services            |
+------------+-------------+
             |
+------------+-------------+
| block and usb drivers    |
+------------+-------------+
             |
+------------+-------------+
| vfs                      |
+------------+-------------+
             |
+------------+-------------+
| apps and services        |
+--------------------------+
```

## 2. RAMFS and VFS Boundary

RAMFS stores named encrypted file records in memory and dispatches open, read,
write, truncate, and close (`userland/capsule_ramfs/src/store/types.rs:23`,
`userland/capsule_ramfs/src/store/types.rs:24`,
`userland/capsule_ramfs/src/store/types.rs:25`,
`userland/capsule_ramfs/src/store/types.rs:26`,
`userland/capsule_ramfs/src/store/types.rs:29`,
`userland/capsule_ramfs/src/store/types.rs:30`,
`userland/capsule_ramfs/src/server/dispatch.rs:27`,
`userland/capsule_ramfs/src/server/dispatch.rs:28`,
`userland/capsule_ramfs/src/server/dispatch.rs:29`,
`userland/capsule_ramfs/src/server/dispatch.rs:30`,
`userland/capsule_ramfs/src/server/dispatch.rs:31`,
`userland/capsule_ramfs/src/server/dispatch.rs:32`).

VFS owns directory/file entries and open FD slots. It caps files, open FDs, and
file bytes, then dispatches open, close, read, write, stat, list, mkdir, unlink,
rename, and healthcheck (`userland/capsule_vfs/src/store/fdtable/types.rs:20`,
`userland/capsule_vfs/src/store/fdtable/types.rs:21`,
`userland/capsule_vfs/src/store/fdtable/types.rs:22`,
`userland/capsule_vfs/src/store/fdtable/types.rs:37`,
`userland/capsule_vfs/src/store/fdtable/types.rs:43`,
`userland/capsule_vfs/src/store/fdtable/types.rs:51`,
`userland/capsule_vfs/src/server/dispatch.rs:27` to
`userland/capsule_vfs/src/server/dispatch.rs:38`).

```
+--------------------------+
| vfs path operation       |
+------------+-------------+
             |
+------------+-------------+
| validate path fd owner   |
+------------+-------------+
             |
+------------+-------------+
| file table mutation      |
+------------+-------------+
             |
+------------+-------------+
| ramfs or vfs response    |
+--------------------------+
```

## 3. Virtio Block

Virtio block allocates receive and transmit buffers sized around read/write
payload limits, receives requests from inbox `0`, decodes the request, and
routes healthcheck, capacity, read blocks, write blocks, and flush
(`userland/capsule_driver_virtio_blk/src/server/runner.rs:25`,
`userland/capsule_driver_virtio_blk/src/server/runner.rs:26`,
`userland/capsule_driver_virtio_blk/src/server/runner.rs:27`,
`userland/capsule_driver_virtio_blk/src/server/runner.rs:31`,
`userland/capsule_driver_virtio_blk/src/server/runner.rs:37`,
`userland/capsule_driver_virtio_blk/src/server/runner.rs:45`,
`userland/capsule_driver_virtio_blk/src/server/runner.rs:46`,
`userland/capsule_driver_virtio_blk/src/server/runner.rs:47`,
`userland/capsule_driver_virtio_blk/src/server/runner.rs:48`,
`userland/capsule_driver_virtio_blk/src/server/runner.rs:49`,
`userland/capsule_driver_virtio_blk/src/server/runner.rs:50`). The protocol op
table declares those five operations (`userland/capsule_driver_virtio_blk/src/protocol/ops.rs:16`,
`userland/capsule_driver_virtio_blk/src/protocol/ops.rs:17`,
`userland/capsule_driver_virtio_blk/src/protocol/ops.rs:18`,
`userland/capsule_driver_virtio_blk/src/protocol/ops.rs:19`,
`userland/capsule_driver_virtio_blk/src/protocol/ops.rs:20`).

## 4. AHCI and NVMe

AHCI polls and acknowledges its IRQ grant, receives fixed-size requests,
rejects payload-bearing requests, then handles healthcheck, controller info, and
port list (`userland/capsule_driver_ahci/src/server/runner.rs:31`,
`userland/capsule_driver_ahci/src/server/runner.rs:38`,
`userland/capsule_driver_ahci/src/server/runner.rs:39`,
`userland/capsule_driver_ahci/src/server/runner.rs:43`,
`userland/capsule_driver_ahci/src/server/runner.rs:46`,
`userland/capsule_driver_ahci/src/server/runner.rs:57`,
`userland/capsule_driver_ahci/src/server/runner.rs:61`,
`userland/capsule_driver_ahci/src/server/runner.rs:62`,
`userland/capsule_driver_ahci/src/server/runner.rs:63`,
`userland/capsule_driver_ahci/src/server/runner.rs:64`).

NVMe polls and acknowledges its IRQ grant, receives fixed-size requests,
rejects payload-bearing requests, then handles healthcheck, controller info,
identify controller, identify namespace, and SMART health
(`userland/capsule_driver_nvme/src/server/runner.rs:30`,
`userland/capsule_driver_nvme/src/server/runner.rs:37`,
`userland/capsule_driver_nvme/src/server/runner.rs:38`,
`userland/capsule_driver_nvme/src/server/runner.rs:49`,
`userland/capsule_driver_nvme/src/server/runner.rs:53`,
`userland/capsule_driver_nvme/src/server/runner.rs:54`,
`userland/capsule_driver_nvme/src/server/runner.rs:55`,
`userland/capsule_driver_nvme/src/server/runner.rs:56`,
`userland/capsule_driver_nvme/src/server/runner.rs:57`,
`userland/capsule_driver_nvme/src/server/runner.rs:58`,
`userland/capsule_driver_nvme/src/server/runner.rs:64`,
`userland/capsule_driver_nvme/src/server/runner.rs:66`,
`userland/capsule_driver_nvme/src/server/runner.rs:68`).

```
+--------------------------+
| ahci or nvme irq poll    |
+------------+-------------+
             |
+------------+-------------+
| ipc request decode       |
+------------+-------------+
             |
+------------+-------------+
| payload length must zero |
+------------+-------------+
             |
+------------+-------------+
| controller query handler |
+--------------------------+
```

## 5. USB Mass Storage

USB MSC owns a binding table, binding count, command tags, probe count, CSW
success and failure counts, phase error count, and residue byte count
(`userland/capsule_driver_usb_msc/src/state/types.rs:20`,
`userland/capsule_driver_usb_msc/src/state/types.rs:21`,
`userland/capsule_driver_usb_msc/src/state/types.rs:22`,
`userland/capsule_driver_usb_msc/src/state/types.rs:23`,
`userland/capsule_driver_usb_msc/src/state/types.rs:24`,
`userland/capsule_driver_usb_msc/src/state/types.rs:25`,
`userland/capsule_driver_usb_msc/src/state/types.rs:26`,
`userland/capsule_driver_usb_msc/src/state/types.rs:27`,
`userland/capsule_driver_usb_msc/src/state/types.rs:28`,
`userland/capsule_driver_usb_msc/src/state/types.rs:29`). Its runner receives
from inbox `0`, parses the request, and passes it to the dispatch layer
(`userland/capsule_driver_usb_msc/src/server/runner.rs:27`,
`userland/capsule_driver_usb_msc/src/server/runner.rs:28`,
`userland/capsule_driver_usb_msc/src/server/runner.rs:30`,
`userland/capsule_driver_usb_msc/src/server/runner.rs:37`,
`userland/capsule_driver_usb_msc/src/server/runner.rs:41`,
`userland/capsule_driver_usb_msc/src/server/runner.rs:42`).

The dispatch layer handles healthcheck, probe config, build inquiry, build read
capacity 10, build read 10, build write 10, accept CSW, and get state
(`userland/capsule_driver_usb_msc/src/server/dispatch.rs:21`,
`userland/capsule_driver_usb_msc/src/server/dispatch.rs:22`,
`userland/capsule_driver_usb_msc/src/server/dispatch.rs:23`,
`userland/capsule_driver_usb_msc/src/server/dispatch.rs:24`,
`userland/capsule_driver_usb_msc/src/server/dispatch.rs:25`,
`userland/capsule_driver_usb_msc/src/server/dispatch.rs:28`,
`userland/capsule_driver_usb_msc/src/server/dispatch.rs:31`,
`userland/capsule_driver_usb_msc/src/server/dispatch.rs:32`,
`userland/capsule_driver_usb_msc/src/server/dispatch.rs:33`,
`userland/capsule_driver_usb_msc/src/server/dispatch.rs:34`).

```
+--------------------------+
| usb msc request          |
+------------+-------------+
             |
+------------+-------------+
| binding state            |
| command tag state        |
+------------+-------------+
             |
+------------+-------------+
| build command or accept  |
| command status wrapper   |
+------------+-------------+
             |
+------------+-------------+
| status snapshot reply    |
+--------------------------+
```

## 6. Failure Map

| Symptom | First source path to inspect | Why |
|---------|------------------------------|-----|
| VFS write fails | `userland/capsule_vfs/src/server/dispatch.rs:31` | Write must reach the VFS write handler after protocol decode. |
| RAMFS handle cannot read | `userland/capsule_ramfs/src/server/dispatch.rs:29` | RAMFS read is a distinct handler from open and write. |
| Virtio block read returns bad status | `userland/capsule_driver_virtio_blk/src/server/runner.rs:48` | Read blocks enters the virtio block read handler from this match arm. |
| AHCI reports no ports | `userland/capsule_driver_ahci/src/server/runner.rs:64` | Port list is the AHCI visible device inventory path. |
| NVMe health data missing | `userland/capsule_driver_nvme/src/server/runner.rs:58` | SMART health is the NVMe health handler. |
| USB MSC command sequence stalls | `userland/capsule_driver_usb_msc/src/server/dispatch.rs:31` | Read and write command builders depend on USB MSC binding and tag state. |

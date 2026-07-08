# The Block Device Layer

Under any on-disk filesystem is a block device, and NØNOS presents one uniform block interface over
three real backends: NVMe, AHCI, and virtio-blk. The kernel side is a thin dispatcher; the drivers
themselves are signed capsules reached over IPC. This page documents the backend selection and the
block operations. The code is under `src/hardware/block_device/`.

## The backend

A `Backend` (`backend.rs:18`) names the three supported controllers, and `selected` (`select.rs:23`)
picks one at first use by probing for capacity in a fixed order:

```
  selected():   // memoized in a Once
      if nvme_capsule::capacity() > 0:  Nvme
      if ahci_capsule::capacity() > 0:  Ahci
      else:                             VirtioBlk
```

NVMe is preferred, then AHCI, then virtio-blk as the fallback that a QEMU guest usually presents.
The choice is made once and cached, so every later operation goes to the same backend. Each of the
three is a real driver, not a stub: NVMe, AHCI, and virtio-blk are all implemented as capsules under
`src/hardware/`, spawned at boot.

## Block operations

The block interface is `read`, `write`, `flush`, `capacity`, and `geometry`, and each dispatches to
the selected backend's capsule client. `read` (`read.rs:24`) is representative:

```
  read(lba, out):
      match selected():
          VirtioBlk => virtio_blk_capsule::read_blocks(lba, out)
          Ahci      => ahci_capsule::read_blocks(lba, out)
          Nvme      => nvme_capsule::read_blocks(lba, out)
```

The call reaches the driver capsule, which owns the controller through the
[hardware broker](../hardware-broker/README.md) and performs the actual device I/O, then copies the
sectors back. This is real hardware access, not an in-memory shim: the block layer is a dispatcher
that forwards a logical block address and a buffer to whichever driver capsule owns the disk. The
geometry is 512-byte sectors (`geometry.rs`), and the errors from each backend are normalized to one
`BlockDeviceError` through the per-backend `map_*_error` functions so a caller sees one error type.

## Where drivers live

The driver capsules are the same kind of ring-3 capsule as everything else: they claim their
controller through the broker, map its registers and set up DMA rings for the queues, and serve block
reads and writes over IPC. The block layer documented here is the kernel-side seam that a filesystem
calls; the driver-side is a capsule. This keeps the disk driver out of the kernel proper, so a bug in
an NVMe or AHCI driver is contained in a capsule rather than being a kernel fault.

## Source

```
  src/hardware/block_device/backend.rs   the Backend enum
  src/hardware/block_device/select.rs    the NVMe -> AHCI -> virtio probe
  src/hardware/block_device/read.rs, write.rs, flush.rs   the dispatched operations
  src/hardware/block_device/geometry.rs, capacity.rs      512-byte geometry and size
  src/hardware/nvme_capsule/, ahci_capsule/, virtio_*     the driver capsules
```

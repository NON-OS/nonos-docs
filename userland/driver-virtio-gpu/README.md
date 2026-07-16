# capsule_driver_virtio_gpu (full reference)

`capsule_driver_virtio_gpu` is the virtio-gpu display-controller driver in the NONOS tree: the signed
userland capsule that owns the PCI device, maps its registers, binds its interrupt, allocates its control
queue DMA, runs the virtio negotiation, creates the primary 2D scanout surface, and then serves an `NVGP`
IPC protocol that the compositor presents through. Everything above the device (composition, damage,
focus, cursor, window ownership) stays outside this capsule; the driver holds the hardware authority and
nothing else. This is the exhaustive reference. The short version lives in the
[drivers overview](../drivers.md) and the client side is documented in the
[compositor reference](../compositor/README.md).

It is a headless service capsule, not an app-skeleton GUI app. Its `_start` runs the setup sequence, then
registers `driver.virtio_gpu0` on service port 4226 and enters a blocking IPC loop
(`userland/capsule_driver_virtio_gpu/src/main.rs:35`). Its capability mask is `0x1F9019`
(`userland/capsule_driver_virtio_gpu/Capsule.mk:16`). The source is
`userland/capsule_driver_virtio_gpu/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [Architecture and bring-up](#architecture-and-bring-up)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The capsule is the 2D display backend for the desktop. On boot it finds a virtio-gpu PCI function, claims
it through the broker, maps BAR0, binds the device interrupt, allocates one broker-owned DMA region for
control queue 0, and drives the virtio ACKNOWLEDGE / DRIVER / FEATURES_OK / DRIVER_OK handshake. It then
asks the device for its display info, records the scanout modes, and builds a primary framebuffer: it
allocates a DMA-backed surface sized to scanout 0, issues `RESOURCE_CREATE_2D`, attaches the backing,
primes the display with a first transfer/scanout/flush, and registers the surface with the kernel so the
compositor can share it (`src/setup/sequence.rs:24`, `src/setup/primary_surface/create.rs:23`).

Once running, the driver serves the `NVGP` IPC protocol. Read-only ops report device and queue state; the
command ops (`GET_PRIMARY_SURFACE`, `CREATE_RESOURCE`, `ATTACH_BACKING`, `TRANSFER_TO_HOST`,
`SET_SCANOUT`, `FLUSH`) let the compositor claim the primary surface and drive real virtio-gpu control
commands onto the device. The compositor resolves `driver.virtio_gpu0` once at setup, fetches its primary
surface, and per frame transfers the dirty rect, sets the scanout on the first frame, and flushes
(`docs/userland/capsule-catalog/compositor.md:297`).

The pixel format the driver creates and enforces everywhere is `VG_FORMAT_B8G8R8A8_UNORM`, value `1`
(`src/constants/mod.rs:68`). There is no VirGL, no Venus, and no 3D acceleration anywhere in this capsule.
The driver negotiates only `VIRTIO_F_VERSION_1` on the modern path and clears all guest feature bits
otherwise (`src/init/modern.rs:40`, `src/init/legacy.rs:30`); it never advertises `VIRTIO_GPU_F_VIRGL`,
and the README lists 3D, virgl, and Venus as explicit non-goals for this slice
(`userland/capsule_driver_virtio_gpu/README.md:131`). The pipeline is 2D-only: the compositor renders in
software and the driver copies finished pixels to the host.

## Identity

Everything the kernel and the service registry need to name and reach the driver comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-virtio-gpu` | `Capsule.mk:5` |
| Service handle | `driver.virtio_gpu0` | `Capsule.mk:6`, `src/hardware/virtio_gpu_capsule/spawn.rs:31` |
| Namespace | `systems.nonos.driver.virtio_gpu0` | `Capsule.mk:11` |
| Service endpoint | `service:4226:driver.virtio_gpu0` | `Capsule.mk:12`, `src/main.rs:32`, `spawn.rs:32` |
| Reply endpoint | `reply:4227:endpoint.4294967316` | `Capsule.mk:13`, `spawn.rs:33`, `spawn.rs:34` |
| Capability mask | `0x1F9019` | `Capsule.mk:16` |
| Binary name | `driver_virtio_gpu` | `Capsule.mk:9` |
| Kernel mirror | `src/hardware/virtio_gpu_capsule` | `Capsule.mk:17` |

The service registers itself by name from `main.rs`: `SERVICE_NAME = b"driver.virtio_gpu0"`,
`SERVICE_PORT = 4226` (`src/main.rs:31`). The reply inbox `endpoint.4294967316` and reply port `4227`
come from the kernel spawn record (`spawn.rs:33`). The reply inbox number `4294967316` is `0x1_0000_0014`,
the reply-endpoint id the spawn machinery assigns; the driver itself replies to each request through
`mk_ipc_reply` addressed to the sender pid, not to a fixed reply port (`src/server/respond.rs:23`).

The mask `0x1F9019` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x000001  CoreExec                bit()       1     types.rs:56
  0x000008  IPC                     bit()       8     types.rs:59
  0x000010  Memory                  bit()      16     types.rs:60
  0x001000  GraphicsSurfaceCreate   bit()    4096     types.rs:68
  0x008000  DeviceEnum              bit()   32768     types.rs:71
  0x010000  Driver                  bit()   65536     types.rs:72
  0x020000  Mmio                    bit()  131072     types.rs:73
  0x040000  Irq                     bit()  262144     types.rs:74
  0x080000  Dma                     bit()  524288     types.rs:75
  0x100000  Pio                     bit() 1048576     types.rs:76
  --------
  0x1F9019  = 1 + 8 + 16 + 4096 + 32768 + 65536 + 131072 + 262144 + 524288 + 1048576
```

The kernel spawn path requests exactly those ten capabilities and no others
(`src/hardware/virtio_gpu_capsule/spawn.rs:50`). There is no `Network` bit, no `FileSystem` bit, and no
`GraphicsDisplayQuery` bit. Unlike an app such as the terminal, this capsule holds the driver-broker
authority quartet (`Driver`, `Mmio`, `Irq`, `Dma`) plus `Pio` and `DeviceEnum`, which is the whole basis
of the security analysis below: it can claim one device and touch that device's registers, interrupt, and
DMA region, and it can create a surface and speak IPC, but it has no filesystem, network, or compositor
authority.

## Operation reference

The IPC protocol is `NVGP`: magic `0x4E56_4750`, version `1`, a 20-byte header, little-endian throughout
(`src/protocol/header.rs:16`). The header is `magic(4) | version(2) | op(2) | flags(2) | reserved(2) |
request_id(4) | payload_len(4)`; the parser rejects any frame whose declared payload length does not
match the buffer (`src/protocol/decode.rs:17`). Every reply repeats the header, then a 4-byte signed
status word, then an optional body; a status-only reply is `HDR(20) | status(4)`, and a payload reply is
`HDR(20) | status(4) | body` with status `0` (`src/server/respond.rs:20`).

Errors are POSIX-style negative ints in the status word (`src/protocol/errno.rs`): `E_INVAL -22`,
`E_NOMEM -12`, `E_BUSY -16`, `E_BAD_OP -38`, `E_DEVICE -110`. The dispatcher replies `E_BAD_OP` to an
unknown op with an empty body and `E_INVAL` to an unknown op that carried a body
(`src/server/runner/dispatch.rs:53`).

The twelve opcodes are defined in `src/protocol/ops.rs:16` and dispatched in
`src/server/runner/dispatch.rs:26`.

### Read-only state ops (empty request body)

| Op | Code | Reply body | Contents | Source |
|---|---|---|---|---|
| `OP_HEALTHCHECK` | `0x0001` | none | status `0` | `ops.rs:16`, `handlers/health.rs:18` |
| `OP_CONTROLLER_INFO` | `0x0002` | 40 bytes | device_id(8), claim_epoch(8), pci_device(2), queue_size(2), host_features(4), mmio_grant(8), irq_grant(8) | `ops.rs:17`, `handlers/controller.rs:19` |
| `OP_DISPLAY_INFO` | `0x0003` | 12 bytes | events_read(4), num_scanouts(4), num_capsets(4), read live from device config | `ops.rs:18`, `handlers/display.rs:19`, `driver.rs:49` |
| `OP_CONTROLQ_STATE` | `0x0004` | 24 bytes | queue_grant(8), queue_user_va(8), queue_device_addr(8) | `ops.rs:19`, `handlers/controlq.rs:19` |
| `OP_QUERY_CAPS` | `0x0005` | 12 bytes | num_scanouts(4), num_capsets(4), events_read(4) | `ops.rs:20`, `handlers/query_caps.rs:19` |
| `OP_MODE_LIST` | `0x000B` | 32 bytes per scanout | id(4), enabled(4), width(4), height(4), x(4), y(4), current_resource_id(4), pad(4); one entry per recorded scanout, bounded by the reply buffer | `ops.rs:26`, `handlers/mode_list.rs:20` |

`OP_DISPLAY_INFO` and `OP_QUERY_CAPS` both re-read the device config MMIO at call time
(`GPU_CFG_EVENTS_READ 0x14`, `GPU_CFG_NUM_SCANOUTS 0x1C`, `GPU_CFG_NUM_CAPSETS 0x20`,
`src/constants/mod.rs:34`, `driver.rs:40`); they differ only in field order. `OP_MODE_LIST` reports the
driver's own recorded scanout table, and its loop refuses to write past the reply buffer even if more
scanouts exist than fit (`handlers/mode_list.rs:28`).

### Command ops

`OP_GET_PRIMARY_SURFACE` takes an empty body; the five that follow carry a fixed-length request body and
post a real virtio-gpu control command onto the device.

| Op | Code | Request body | Reply | Errors | Source |
|---|---|---|---|---|---|
| `OP_GET_PRIMARY_SURFACE` | `0x000C` | none | 32 bytes: handle(8), resource_id(4), width(4), height(4), stride(4), format(4), pad(4) | `E_DEVICE` if no primary, `E_BUSY` if owned by another pid | `ops.rs:27`, `handlers/get_primary_surface.rs:22` |
| `OP_CREATE_RESOURCE` | `0x0006` | 16 bytes: requested_id(4), format(4), width(4), height(4) | 4 bytes: resource_id | `E_INVAL` on bad length/zero dims/wrong format, `E_DEVICE` on device reject, `E_NOMEM` if the table is full | `ops.rs:21`, `handlers/create_resource.rs:24` |
| `OP_ATTACH_BACKING` | `0x0007` | 24 bytes: resource_id(4), pad(4), backing_addr(8), backing_len(8) | status only | `E_INVAL` on bad args or unknown resource, `E_BUSY` if not the owner, `E_DEVICE` on device reject | `ops.rs:22`, `handlers/attach_backing.rs:20` |
| `OP_TRANSFER_TO_HOST` | `0x0008` | 32 bytes: resource_id(4), x(4), y(4), width(4), height(4), pad(4), offset(8) | status only | `E_INVAL` on bad rect or unknown resource, `E_BUSY` if not the owner, `E_DEVICE` on device reject | `ops.rs:23`, `handlers/transfer_to_host.rs:24` |
| `OP_SET_SCANOUT` | `0x0009` | 24 bytes: scanout_id(4), resource_id(4), x(4), y(4), width(4), height(4) | status only | `E_INVAL` on bad args, wrong format, or unknown resource, `E_BUSY` if not the owner, `E_DEVICE` on device reject | `ops.rs:24`, `handlers/set_scanout.rs:24` |
| `OP_FLUSH` | `0x000A` | 20 bytes: resource_id(4), x(4), y(4), width(4), height(4) | status only | `E_INVAL` on bad rect or unknown resource, `E_BUSY` if not the owner, `E_DEVICE` on device reject | `ops.rs:25`, `handlers/flush.rs:20` |

These are the exact opcodes the compositor's gfx client uses. It calls `GET_PRIMARY_SURFACE` (`0x000C`)
once at setup, then per frame `TRANSFER_TO_HOST` (`0x0008`), `SET_SCANOUT` (`0x0009`) on the first frame
only, and `RESOURCE_FLUSH` (`0x000A`) (`docs/userland/capsule-catalog/compositor.md:297`). The opcodes
line up byte for byte.

Two invariants govern the command ops. First, `GET_PRIMARY_SURFACE` claims ownership: the first caller to
ask becomes the resource's `owner_pid`, and thereafter every command op checks `owner_pid == sender_pid`
and returns `E_BUSY` to any other pid (`handlers/get_primary_surface.rs:27`,
`handlers/transfer_to_host.rs:33`). Second, every rect is validated against the resource's own width and
height with saturating arithmetic before the device command is issued, so a client cannot ask the device
to read outside the resource (`handlers/transfer_to_host.rs:37`, `handlers/flush.rs:53`). `ATTACH_BACKING`
additionally checks that the requested backing address range lies inside the primary surface's own
broker-granted DMA region, refusing any range the capsule does not own (`handlers/attach_backing.rs:49`,
`:72`).

## Architecture and bring-up

The capsule is `no_std`/`no_main`. `_start` initializes the heap, then loops calling `setup::run()` until
the device comes up, yielding between attempts, then registers the service and enters the server
(`src/main.rs:35`). The top-level modules are `discover`, `setup`, `init`, `regs`, `device`, `state`,
`driver`, `protocol`, and `server` (`src/main.rs:19`).

The setup sequence is the whole bring-up, in order (`src/setup/sequence.rs:24`):

1. Discover a virtio-gpu PCI function. `find_virtio_gpu` lists devices via `mk_device_list`, matches
   vendor `0x1AF4` with device id `0x1010` (transitional) or `0x1050` (modern), and requires a usable IRQ
   pin and line (`src/discover/search.rs:25`, `src/discover/match_device.rs:21`). It selects the register
   BAR: for the modern id it prefers an MMIO BAR in the `0x4000..0x10000` config window, else the first
   MMIO or PIO BAR (`src/discover/bar_select.rs:24`).
2. Claim the device through the broker: `mk_device_claim` returns a `claim_epoch` that authorizes every
   later grant on this device (`src/setup/claim.rs:17`).
3. Enable PCI bus mastering so the device can DMA: `mk_pci_config_write` sets the bus-master command bit,
   releasing the device on failure (`src/setup/pci.rs:20`).
4. Map the register BAR. The modern id maps its modern capability window; otherwise an MMIO BAR is mapped
   page-rounded with `mk_mmio_map` at `BAR_OFFSET 0`, or a PIO BAR is granted with a port-IO grant
   (`src/setup/mmio/grant.rs:23`, `src/setup/mmio/map_mmio.rs:22`).
5. Bind the interrupt. `mk_irq_bind` tries INTx on the device's IRQ line first, then MSI-X; a device with
   no line yields a zero grant rather than an error (`src/setup/irq.rs:19`).
6. Allocate the control queue DMA. `mk_dma_map` allocates the `VQ_REGION_SIZE` (16 KiB) queue region; on
   failure it rolls back the IRQ bind, the register map, and the device claim in reverse order
   (`src/setup/dma.rs:19`, `src/constants/mod.rs:20`).
7. Run the virtio negotiation. `bring_up` picks the modern or legacy path by device id
   (`src/init/bring_up.rs:23`). Both write STATUS_ACKNOWLEDGE then STATUS_DRIVER, read host features,
   clear or set only the version-1 feature bit, set STATUS_FEATURES_OK and verify the device accepted it,
   program the control queue, and finish with STATUS_DRIVER_OK (`src/init/legacy.rs:24`,
   `src/init/modern.rs:28`). A zero queue size or a rejected FEATURES_OK sets STATUS_FAILED and aborts.

The queue region is laid out at fixed offsets inside the 16 KiB DMA region: the descriptor table at 0, the
available ring at 4096, the used ring at 8192, and a 4 KiB staging area at 12288
(`src/constants/mod.rs:22`). The control queue is a standard split virtqueue driven synchronously. Each
command writes its request bytes into staging, then a zeroed response tail, builds a two-descriptor chain
(a read segment with `VRING_DESC_F_NEXT` and a device-writable response segment with `VRING_DESC_F_WRITE`),
publishes the head in the available ring, notifies the device, waits for the used ring to advance,
verifies the used id matches the head, and reads the response back out of staging
(`src/device/virtqueue/submit.rs:23`, `src/device/virtqueue/desc.rs:29`). Every command carries a monotonic
fence id from a `FenceCounter` (`src/state/fences.rs`, issued in each handler).

The scanout/framebuffer model. After DRIVER_OK the driver seeds its scanout table from the device's
`GET_DISPLAY_INFO` (virtio command `0x0100`), recording each enabled scanout; a scanout smaller than
1280x720 is promoted to the 1920x1080 default, and if the device reports none it seeds a single default
scanout 0 (`src/setup/scanouts.rs:24`). It then builds the primary surface for scanout 0
(`src/setup/create_primary.rs:21`): derive the geometry (stride = width times 4, byte length =
stride times height, capped at `u32::MAX`, `src/setup/primary_surface/geometry.rs:22`), map a page-rounded
high DMA region for the backing store (`src/setup/primary_surface/dma.rs:19`), allocate a resource id,
issue `RESOURCE_CREATE_2D` (`0x0101`) and `RESOURCE_ATTACH_BACKING` (`0x0106`), then prime the display
with `TRANSFER_TO_HOST_2D` (`0x0105`), `SET_SCANOUT` (`0x0103`), and `RESOURCE_FLUSH` (`0x0104`)
(`src/setup/primary_surface/create.rs:37`, `src/setup/primary_surface/prime.rs:22`). Finally it registers
the surface with the kernel via `mk_surface_register` and shares it via `mk_surface_share`, storing the
resulting handle; any failure rolls back the DMA grant (`src/setup/primary_surface/create.rs:48`). The
DMA path is what makes this a real backend: the backing store is a broker-owned physical region the device
reads by physical address, and the transfer/flush commands copy the compositor's finished pixels from that
region to the host framebuffer.

The device commands map to the virtio-gpu control opcodes (`src/constants/mod.rs:60`):
`GET_DISPLAY_INFO 0x0100`, `RESOURCE_CREATE_2D 0x0101`, `SET_SCANOUT 0x0103`, `RESOURCE_FLUSH 0x0104`,
`TRANSFER_TO_HOST_2D 0x0105`, `RESOURCE_ATTACH_BACKING 0x0106`, with `RESP_OK_NODATA 0x1100` and
`RESP_OK_DISPLAY_INFO 0x1101` as the accepted responses (`src/device/cmd/create_resource_2d.rs:42`,
`src/device/cmd/get_display_info.rs:45`). Every command builds its request with a 24-byte control header
(`type, flags, fence_id, ctx_id`, `src/device/cmd/hdr.rs:16`) and checks the returned type before
reporting success.

## Protocol and IPC

Inbound, the server loop receives on the service inbox with `mk_ipc_recv_from`, ignores empty frames or a
zero sender pid, parses the `NVGP` header, and dispatches (`src/server/runner/loop_once.rs:25`). The
receive and reply buffers are each `HDR_LEN + IPC_PAYLOAD_MAX` = 276 bytes (`src/server/runner/entry.rs:23`,
`src/protocol/limits.rs:16`). Replies go back to the sender pid via `mk_ipc_reply`
(`src/server/respond.rs:23`).

The driver reaches hardware only through broker syscalls, one per authority bit
(`userland/capsule_driver_virtio_gpu/README.md:29`):

```
  mk_device_list        enumerate PCI (DeviceEnum)          discover/search.rs:27
  mk_device_claim       claim the device (Driver)           setup/claim.rs:18
  mk_pci_config_write   set bus-master (Driver)             setup/pci.rs:21
  mk_mmio_map           map BAR0 (Mmio)                     setup/mmio/map_mmio.rs:25
  mk_pio_grant          grant a PIO window (Pio)            setup/mmio/grant_pio.rs
  mk_irq_bind / _ack    bind and ack INTx/MSI-X (Irq)       setup/irq.rs:28, sequence.rs:35
  mk_dma_map / _unmap   allocate the queue + surface (Dma)  setup/dma.rs:26, primary_surface/dma.rs:24
  mk_surface_register   register the primary surface        primary_surface/create.rs:48
  mk_surface_share      share the surface handle            primary_surface/create.rs:53
  mk_service_register   register driver.virtio_gpu0         main.rs:50
  mk_ipc_recv_from / _reply   serve the NVGP protocol       server/runner/loop_once.rs:28, respond.rs:23
```

The broker owns each grant, validates it against the claim epoch, and tears it down on capsule exit. The
`ATTACH_BACKING` handler is the one place a client-supplied physical address reaches a device command, and
it is bounded before it does: the address range must fall inside the primary surface's own DMA region or
the driver rejects it with `E_INVAL` (`handlers/attach_backing.rs:49`). The four broker facets are
documented in `docs/subsystems/hardware-broker/{claim,mmio,dma,irq}.md`.

## Security analysis

This capsule holds real hardware authority, so its bounds matter more than an app's. The mask `0x1F9019`
grants CoreExec, IPC, Memory, GraphicsSurfaceCreate, DeviceEnum, Driver, Mmio, Irq, Dma, and Pio
(`Capsule.mk:16`, `src/hardware/virtio_gpu_capsule/spawn.rs:50`). There is no Network, no FileSystem, no
compositor, and no input authority. The driver can claim exactly one virtio-gpu PCI function, map that
device's register BAR, bind that device's interrupt, allocate broker-owned DMA for its control queue and
its primary framebuffer, and mint a PIO window on the legacy path. It cannot read a block device, open a
socket, touch another device's registers, or composite.

The broker is the enforcement point for all of it. Each grant is scoped to the device and the claim epoch
`mk_device_claim` returns, so a grant is only valid for the one device the capsule legitimately claimed
(`src/setup/claim.rs:17`). Register mapping is page-rounded to the requested BAR (`map_mmio.rs:24`); the
DMA regions are sized exactly (16 KiB for the queue, the page-rounded surface length for the framebuffer)
and are broker-owned, so they are revoked on exit (`src/setup/dma.rs:26`, `primary_surface/dma.rs:20`).
Every setup phase rolls back the grants it already holds if a later phase fails, so a partial bring-up
never leaves a device claimed or an IRQ bound (`src/setup/dma.rs:27`).

The honest caveat is the IOMMU. The broker returns a raw physical `device_addr` and the shipping builds do
not engage an IOMMU backend, so a device programmed with a physical address can in principle DMA anywhere
regardless of the grant; the broker bounds what a capsule may allocate, not where a compliant device
actually reads (`docs/subsystems/hardware-broker/dma.md:93`). This capsule narrows that surface as far as
software can: it clears all optional virtio feature bits, drives only the fixed control commands it builds
itself, and refuses any `ATTACH_BACKING` address outside the surface region it owns
(`handlers/attach_backing.rs:49`). It cannot hand the device an arbitrary guest address through IPC. The
residual trust is in the virtio-gpu device honoring the descriptor bounds, which is the same trust every
DMA-capable driver carries until the IOMMU backend is enabled.

Client isolation is by ownership. The primary resource is claimed by the first pid to call
`GET_PRIMARY_SURFACE`, and every subsequent command op enforces `owner_pid == sender_pid`, replying
`E_BUSY` to anyone else (`handlers/get_primary_surface.rs:27`, `handlers/set_scanout.rs:33`). A second
capsule cannot transfer, scanout, or flush a resource it does not own. Isolation from other capsules is
the kernel's: the driver is a CPL 3 user binary that speaks only IPC and its granted hardware, and it is
verified and enrolled at spawn like every other capsule (`src/hardware/virtio_gpu_capsule/spawn.rs:62`).

## How to contribute

The source lives at `userland/capsule_driver_virtio_gpu/`. Discovery is under `src/discover/`, the brokered
bring-up under `src/setup/`, the virtio negotiation under `src/init/`, the register accessors under
`src/regs/`, the virtqueue and control commands under `src/device/`, the runtime tables under `src/state/`,
the wire format under `src/protocol/`, and the IPC server under `src/server/`.

To add a new IPC op:

1. Define the opcode in `src/protocol/ops.rs:16` and any fixed request/response length in
   `src/protocol/limits.rs:16`, then re-export both from `src/protocol/mod.rs:33`.
2. Write the handler as one file under `src/server/handlers/`, exposing
   `pub fn handle(driver: &Driver, sender_pid: u32, req: &Request, body: &[u8], tx: &mut [u8])` (drop
   `body` for empty-body ops). Validate the body length first and reply `E_INVAL` on a mismatch, then
   enforce resource ownership before issuing any device command, following the shape of
   `handlers/flush.rs`. Register the module in `src/server/handlers/mod.rs:16`.
3. Wire the opcode into the match in `src/server/runner/dispatch.rs:26`, using the `if body.is_empty()`
   guard for empty-body ops the way the read-only ops are wired.
4. If the op posts a new virtio-gpu command, add the command builder as one file under
   `src/device/cmd/` and re-export it from `src/device/cmd/mod.rs:16`, using the 24-byte control header
   and checking the response type against the expected `RESP_OK_*` constant.

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_driver_virtio_gpu/Capsule.mk:19`):

```
  make nonos-mk-driver-virtio-gpu            build the capsule ELF
  make nonos-mk-driver-virtio-gpu-sign       produce the id cert, manifest, and attestation trailer
  make nonos-mk-driver-virtio-gpu-verify     verify the signed artifacts against the trust anchor
  make nonos-mk-check-driver-virtio-gpu-keys check the per-capsule signing keys exist
```

For a running kernel that embeds the driver, `make nonos-mk-driver-virtio-gpu-prod` builds the
`microkernel-driver-virtio-gpu` kernel profile with the signed artifacts (`Makefile:945`). The README also
documents a direct build with `make -B nonos-mk-driver-virtio-gpu` and a profile check with
`cargo check --no-default-features --features microkernel-driver-virtio-gpu`
(`userland/capsule_driver_virtio_gpu/README.md:138`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns errors as a negative status word, and the setup path
returns `Result<_, &'static str>` and retries rather than panicking, `src/main.rs:39`); modular files, one
unit per file, with `mod.rs` used only for re-exports; and the AGPL header at the top of every source
file, matching the header on every existing module.

## Debugging

The first thing to confirm is that the capsule was spawned. The kernel embeds and verifies the signed ELF,
cert, manifest, and attestation, then spawns it under `driver.virtio_gpu0` on port 4226 through
`spawn_driver_virtio_gpu_capsule` (`src/hardware/virtio_gpu_capsule/spawn.rs:37`). If the embed feature
`nonos-capsule-driver-virtio-gpu` is off, the embedded ELF is an empty slice and there is nothing to spawn
(`src/hardware/virtio_gpu_capsule/embed.rs:34`), so build the `-prod` profile to include it.

Failure modes and where to look:

- No display, capsule never serves. Bring-up is a retry loop: if `setup::run` keeps failing, the service
  is never registered and no client can resolve `driver.virtio_gpu0` (`src/main.rs:39`). The setup errors
  are specific static strings: `virtio-gpu: device not found` (no matching PCI function or no usable IRQ,
  `src/setup/sequence.rs:26`, `src/discover/match_device.rs:27`), `virtio-gpu: claim failed`
  (`src/setup/claim.rs:20`), `virtio-gpu: features rejected` or `virtio-gpu: missing control queue` (the
  virtio handshake, `src/init/legacy.rs:34`, `:40`), and the DMA/register rollback strings on a failed
  queue map (`src/setup/dma.rs:30`). Confirm the guest was started with a `virtio-gpu-pci` device.
- Blank scanout, capsule serves but nothing is presented. The driver only copies pixels the compositor
  transfers; it does not draw. If `GET_PRIMARY_SURFACE` returns `E_DEVICE`, the primary surface was never
  built, usually a failed display-info or a failed surface registration during setup
  (`handlers/get_primary_surface.rs:24`, `src/setup/primary_surface/create.rs:48`). If it returns `E_BUSY`,
  another pid already owns the primary resource (`handlers/get_primary_surface.rs:39`). If the compositor's
  first-frame `SET_SCANOUT` is rejected, the resource is never bound to the panel and the screen stays
  blank (`docs/userland/capsule-catalog/compositor.md:419`).
- A command op returns `E_INVAL` or `E_BUSY`. `E_INVAL` is a bad body length, a zero-area or
  out-of-bounds rect, a wrong pixel format (only `B8G8R8A8_UNORM`), or an unknown resource id
  (`handlers/create_resource.rs:45`, `handlers/transfer_to_host.rs:39`). `E_BUSY` means the caller is not
  the resource owner (`handlers/set_scanout.rs:33`). `E_DEVICE` means the device rejected the control
  command, so the used-ring response type was not the expected `RESP_OK_*`
  (`src/device/cmd/set_scanout.rs:44`).

## Source map

```
  src/main.rs                              _start -> setup::run -> service register -> server::run
  src/discover/                            PCI enumeration, vendor/device match, register BAR select
  src/setup/sequence.rs                    the full bring-up (claim, bus-master, map, irq, dma, negotiate, primary)
  src/setup/{claim,pci,mmio,irq,dma}.rs    the brokered device claim, register map, irq bind, queue dma
  src/setup/primary_surface/               the DMA-backed primary framebuffer and surface register/share
  src/setup/scanouts.rs                    seed the scanout table from GET_DISPLAY_INFO
  src/init/{legacy,modern}.rs              the virtio ACK/DRIVER/FEATURES_OK/DRIVER_OK negotiation
  src/regs/                                the MMIO/PIO register accessors and device config reads
  src/device/virtqueue/                    the split virtqueue: desc/avail/used rings, submit, wait
  src/device/cmd/                          the virtio-gpu control commands (create/attach/transfer/scanout/flush/display-info)
  src/state/                               the resource table, scanout table, and fence counter
  src/driver.rs                            the Driver struct: grants, queue addrs, config reads
  src/protocol/                            the NVGP wire format: header, ops, limits, errno, decode/encode
  src/server/runner/                       the IPC receive loop and opcode dispatch
  src/server/handlers/                     one file per op
  Capsule.mk                               slug, handle, ports, capability mask, kernel mirror
  src/hardware/virtio_gpu_capsule          the kernel-side embed and verified spawn
  nonos-mk/capsule.mk                      the generated nonos-mk-driver-virtio-gpu[-sign|-verify] targets
```

Every reference above is verified against those trees.

# The virtio-gpu Driver Capsule

`capsule_driver_virtio_gpu` is the 2D display backend the desktop compositor presents through. It is a
signed userland capsule that owns one virtio-gpu PCI function: it claims the device through the broker,
maps its registers, binds its interrupt, allocates its control-queue DMA, runs the virtio negotiation,
builds the primary scanout surface, and then serves an `NVGP` IPC protocol that the compositor drives per
frame. Everything above the device (composition, damage, focus, cursor, window ownership) stays outside
this capsule; the driver holds the hardware authority and nothing else.

The source is organized into three engine pillars plus a wire layer, and this documentation mirrors that
structure one page per pillar so a page can be read beside the folder it describes.

The honest 2D-versus-3D split is the load-bearing fact and it is the opposite of what an earlier version
of this page claimed. The **2D scanout path is the proven, shipping path**: the capsule builds one
`B8G8R8A8_UNORM` primary surface at boot (`src/setup/primary_surface/create.rs:37`,
`prime.rs:28`) and the compositor drives `TRANSFER_TO_HOST_2D` / `SET_SCANOUT` / `RESOURCE_FLUSH` onto it
per frame; this is what puts the desktop on screen. A **full VirGL/3D command stream is also built into
this capsule but is not used by anything in the shipping path**. `src/virgl/` is a complete Gallium stream
builder, `src/constants/cmd_3d.rs:19` defines the 3D opcode set (`VIRTIO_GPU_F_VIRGL`, `CTX_CREATE 0x0200`,
`RESOURCE_CREATE_3D 0x0204`, `SUBMIT_3D 0x0207`), and `src/device/cmd/` carries the 3D command implementations
(`ctx_create.rs`, `submit_3d.rs`, `resource_create_3d/`, `transfer_3d/`). At boot, when the modern transport
negotiates the VirGL feature (`src/init/modern.rs:44`; legacy transport forces it off, `init/legacy.rs:44`),
the driver runs a one-shot self-test that creates a render context, submits a real `SUBMIT_3D` clear, reads
it back, and tears the resource down (`src/setup/probe_3d/run.rs:26`), storing the result as `virgl_ready`
(`src/setup/sequence.rs:46`) and exposing it only through the `QUERY_CAPS` IPC op
(`src/server/handlers/query_caps.rs:27`). No shipping capsule ever issues `SUBMIT_3D`: the compositor
composites in software and presents through the 2D ops exclusively. The 3D path is therefore
**IMPLEMENTED and boot-probed, but NOT USED**, and it only means anything against a host virglrenderer
backend behind a `virtio-vga-gl` device; the guest never renders, it builds streams for host-side
execution (`src/constants/cmd_3d.rs:16`). Treat it as dead in the desktop until a client is written to it.

There is no `GET_EDID` command anywhere in the capsule. Panel size is taken from `GET_DISPLAY_INFO`
(`0x0100`, `src/device/cmd/get_display_info.rs:35`), seeded at scanout setup
(`src/setup/scanouts.rs:29`), and when firmware reports nothing usable the driver falls back to a hardcoded
1280x720 (`scanouts.rs:20`). EDID-based native-resolution sizing is not implemented.

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
| Capability mask | `0x1F9119` | `Capsule.mk:20` |
| Binary name | `driver_virtio_gpu` | `Capsule.mk:9` |
| Kernel mirror | `src/hardware/virtio_gpu_capsule` | `Capsule.mk:17` |

The service registers itself by name from `main.rs`: `SERVICE_NAME = b"driver.virtio_gpu0"`,
`SERVICE_PORT = 4226` (`src/main.rs:31`, `:32`). The reply inbox `endpoint.4294967316` and reply port
`4227` come from the kernel spawn record (`spawn.rs:33`). The inbox number `4294967316` is `0x1_0000_0014`,
the reply-endpoint id the spawn machinery assigns; the driver itself replies to each request through
`mk_ipc_reply` addressed to the sender pid, not to a fixed reply port (`src/server/respond.rs:23`).

The mask `0x1F9119` decomposes bit by bit against `src/capabilities/types/bit.rs` (the enum lives in
`src/capabilities/types/defs.rs`):

```
  0x000001  CoreExec                bit()       1     types/bit.rs:23
  0x000008  IPC                     bit()       8     types/bit.rs:26
  0x000010  Memory                  bit()      16     types/bit.rs:27
  0x000100  Debug                   bit()     256     types/bit.rs:31
  0x001000  GraphicsSurfaceCreate   bit()    4096     types/bit.rs:35
  0x008000  DeviceEnum              bit()   32768     types/bit.rs:38
  0x010000  Driver                  bit()   65536     types/bit.rs:39
  0x020000  Mmio                    bit()  131072     types/bit.rs:40
  0x040000  Irq                     bit()  262144     types/bit.rs:41
  0x080000  Dma                     bit()  524288     types/bit.rs:42
  0x100000  Pio                     bit() 1048576     types/bit.rs:43
  --------
  0x1F9119  = 1 + 8 + 16 + 256 + 4096 + 32768 + 65536 + 131072 + 262144 + 524288 + 1048576
```

The kernel spawn path requests exactly those eleven capabilities and no others
(`src/hardware/virtio_gpu_capsule/spawn.rs:50`), where the `Debug` bit is granted through
`serial_debug_cap()` (`spawn.rs:53`). The `Debug` bit was added to this manifest late: it was originally
missing, so the kernel's serial-debug grant fell outside the manifest ceiling and the spawn gate rejected
the driver, leaving the compositor with no GPU backend (`Capsule.mk:16`). There is no `Network` bit, no
`FileSystem` bit, and no `GraphicsDisplayQuery` bit. Unlike an app such as the terminal, this capsule holds
the driver-broker authority quartet (`Driver`, `Mmio`, `Irq`, `Dma`) plus `Pio` and `DeviceEnum`. That is
the whole basis of its trust boundary: it can claim one device and touch that device's registers,
interrupt, and DMA region, and it can create a surface and speak IPC, but it has no filesystem, network, or
compositor authority. Compromising the driver yields the driver's mask and one virtio-gpu function, nothing
more.

## The three pillars

The source under `userland/capsule_driver_virtio_gpu/src/` is a wire layer over three engine pillars, and
the documentation is one page each. Data flows top to bottom at boot and left to right per frame: bring-up
stands the device up, the engine drives the split virtqueue and holds the resource and scanout tables, and
the client/protocol layer turns compositor IPC into engine calls.

```
  discover/ + setup/ + init/   ->   device/ + state/ + regs/   ->   protocol/ + server/
  find, claim, grant, negotiate     virtqueue, control cmds,       NVGP wire, dispatch,
  the primary surface               resource/scanout tables        the 12 handlers
```

| Page | Mirrors | What it covers |
|---|---|---|
| [bring-up.md](bring-up.md) | `src/discover/`, `src/setup/`, `src/init/` | Finding the PCI function, the brokered claim/bus-master/map/irq/dma quartet, the virtio ACK/DRIVER/FEATURES_OK/DRIVER_OK negotiation, seeding scanouts, and building the DMA-backed primary surface. |
| [engine.md](engine.md) | `src/device/`, `src/state/`, `src/regs/`, `src/driver.rs` | The split virtqueue, the six virtio-gpu control commands, the register accessors, and the resource, scanout, and fence tables the handlers act on. |
| [client-protocol.md](client-protocol.md) | `src/protocol/`, `src/server/` | The `NVGP` wire format, the receive loop and dispatcher, the twelve ops, primary-surface ownership, and the rect and backing-address bounds every command op enforces. |
| [contributing.md](contributing.md) | the whole tree | Where to work, how to add an IPC op or a device command, the build and sign steps, and the code standards. |
| [debugging.md](debugging.md) | runtime | The spawn markers, the no-display and blank-scanout failure modes, and where each `E_*` status comes from. |

## Lifecycle

The capsule is `no_std`/`no_main`. Its `_start` initializes the heap, then loops calling `setup::run()`
until the device comes up, yielding between attempts; on success it registers `driver.virtio_gpu0` on
service port 4226 and enters a blocking IPC loop (`src/main.rs:35`). The kernel embeds and verifies the
signed ELF, cert, manifest, and attestation before mapping it, and requests exactly the ten capabilities
above (`src/hardware/virtio_gpu_capsule/spawn.rs:37`, `:50`); it is a CPL 3 user binary like every other
capsule. The [bring-up](bring-up.md) page walks the setup sequence, and the [debugging](debugging.md) page
covers what a failed bring-up looks like on the boot log.

Once running, read-only ops report device and queue state, and the six command ops let the compositor
claim the primary surface and drive real virtio-gpu control commands onto the device. The compositor
resolves `driver.virtio_gpu0` once at setup, fetches its primary surface, and per frame transfers the
dirty rect, sets the scanout on the first frame, and flushes; the client side is documented in the
[compositor gpu-client reference](../compositor/gpu-client.md).

## Source map

```
  userland/capsule_driver_virtio_gpu/src/main.rs   _start -> setup::run -> service register -> server::run
  userland/capsule_driver_virtio_gpu/src/discover/ src/setup/ src/init/   the bring-up pillar
  userland/capsule_driver_virtio_gpu/src/device/ src/state/ src/regs/ src/driver.rs   the engine pillar
  userland/capsule_driver_virtio_gpu/src/protocol/ src/server/   the client/protocol pillar
  userland/capsule_driver_virtio_gpu/Capsule.mk    slug, handle, ports, capability mask, kernel mirror
  src/capabilities/types/bit.rs                    the capability bit values
  src/hardware/virtio_gpu_capsule                  the kernel-side embed and verified spawn
```

Everything here is drawn from `userland/capsule_driver_virtio_gpu/` (the capsule source and its
`Capsule.mk`), `src/capabilities/types/bit.rs` (the capability bit values), and the kernel spawn mirror under
`src/hardware/virtio_gpu_capsule/`. Every reference above is verified against those trees.

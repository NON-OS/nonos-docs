# Driver Capsules

This page documents the user-mode hardware driver capsules. Read
[Capsule Inventory](capsules.md), [Hardware Broker](../subsystems/hardware-broker/README.md),
[Input](../subsystems/input/README.md), [Graphics](../subsystems/graphics/README.md), and
[Storage](../subsystems/storage/README.md) first.

Read each driver as two contracts. The first contract is its service surface:
which port, capability set, and protocol operations it exposes. The second is
its side effect on the system: packets, blocks, display state, or input events.

---

## 1. Boot group

Driver startup is split into virtio, bus, input, NIC, USB, and storage groups
(`src/userspace/init/spawn_plan/orchestrator.rs:29`). The virtio group delegates
I/O drivers and display/network drivers separately
(`src/userspace/init/spawn_plan/drivers_virtio.rs:17`). USB, NIC, bus, input,
and storage groups each call their capsule spawn functions in fixed order
(`src/userspace/init/spawn_plan/drivers_usb.rs:17`,
`src/userspace/init/spawn_plan/drivers_nic.rs:17`,
`src/userspace/init/spawn_plan/drivers_bus.rs:17`,
`src/userspace/init/spawn_plan/drivers_input.rs:17`,
`src/userspace/init/spawn_plan/drivers_storage.rs:17`).

```
+----------------+
| init drivers   |
+-------+--------+
        |
+-------+--------+
| virtio group   |
+-------+--------+
        |
+-------+--------+
| bus group      |
+-------+--------+
        |
+-------+--------+
| input group    |
+-------+--------+
        |
+-------+--------+
| nic group      |
+-------+--------+
        |
+-------+--------+
| usb group      |
+-------+--------+
        |
+-------+--------+
| storage group  |
+----------------+
```

## 2. Driver contract table

Each of the thirteen non-network drivers has a dedicated page in its own
`driver-<name>/` folder here (for example [driver-nvme](driver-nvme/README.md))
with the full operation reference, bring-up, and source map; the `Page` column
links it. The five network drivers
(`e1000`, `iwlwifi`, `rtl8139`, `rtl8169`, `virtio_net`) are documented by the
[networking subsystem](../subsystems/networking/README.md) and have no dedicated
capsule page. Masks below are the signed `CAPSULE_REQUIRED_CAPS` from each
capsule's `Capsule.mk`.

| Capsule | Service | Caps | Protocol operations | Entrypoint | Page | Spec refs |
|---------|---------|------|---------------------|------------|------|-----------|
| `driver.virtio_rng` | `service:4200:driver.virtio_rng` | `0x1F8019` | fill random, healthcheck | `userland/capsule_driver_virtio_rng/src/main.rs:35` | [driver-virtio-rng](driver-virtio-rng/README.md) | `userland/capsule_driver_virtio_rng/Capsule.mk:13`, `userland/capsule_driver_virtio_rng/Capsule.mk:17`, `userland/capsule_driver_virtio_rng/src/protocol/ops.rs:21` |
| `driver.virtio_blk0` | `service:4202:driver.virtio_blk0` | `0x1F8119` | healthcheck, capacity, read blocks, write blocks, flush | `userland/capsule_driver_virtio_blk/src/main.rs:30` | [driver-virtio-blk](driver-virtio-blk/README.md) | `userland/capsule_driver_virtio_blk/Capsule.mk:13`, `userland/capsule_driver_virtio_blk/Capsule.mk:16`, `userland/capsule_driver_virtio_blk/src/protocol/ops.rs:16` |
| `driver.virtio_net0` | `service:4204:driver.virtio_net0` | `0x1F8119` | healthcheck, link status, MAC address, TX packet, RX packet | `userland/capsule_driver_virtio_net/src/main.rs:36` | [networking](../subsystems/networking/README.md) | `userland/capsule_driver_virtio_net/Capsule.mk:14`, `userland/capsule_driver_virtio_net/Capsule.mk:17`, `userland/capsule_driver_virtio_net/src/protocol/ops.rs:21` |
| `driver.virtio_gpu0` | `service:4226:driver.virtio_gpu0` | `0x1F9119` | healthcheck, controller info, display info, controlq state, query caps, create resource, attach backing, transfer to host, set scanout, flush, mode list, primary surface | `userland/capsule_driver_virtio_gpu/src/main.rs:35` | [driver-virtio-gpu](driver-virtio-gpu/README.md) | `userland/capsule_driver_virtio_gpu/Capsule.mk:12`, `userland/capsule_driver_virtio_gpu/Capsule.mk:16`, `userland/capsule_driver_virtio_gpu/src/protocol/ops.rs:16` |
| `driver.xhci0` | `service:4206:driver.xhci0` | `0xF8019` | healthcheck, controller status, port status, enable slot, disable slot, address device, device descriptor, config descriptor, transfer ring allocation, control transfer, interrupt in | `userland/capsule_driver_xhci/src/main.rs:36` | [driver-xhci](driver-xhci/README.md) | `userland/capsule_driver_xhci/Capsule.mk:13`, `userland/capsule_driver_xhci/Capsule.mk:16`, `userland/capsule_driver_xhci/src/protocol/ops.rs:16` |
| `driver.ps2_kbd0` | `service:4208:driver.ps2_kbd0` | `0x358019` | healthcheck, poll events, get state, controller status, poll mouse | `userland/capsule_driver_ps2_input/src/main.rs:31` | [driver-ps2-input](driver-ps2-input/README.md) | `userland/capsule_driver_ps2_input/Capsule.mk:13`, `userland/capsule_driver_ps2_input/Capsule.mk:17`, `userland/capsule_driver_ps2_input/src/protocol/ops.rs:16` |
| `driver.e1000_0` | `service:4210:driver.e1000_0` | `0xF8039` | healthcheck, link status, MAC address, TX packet, RX packet, stats | `userland/capsule_driver_e1000/src/main.rs:38` | [networking](../subsystems/networking/README.md) | `userland/capsule_driver_e1000/Capsule.mk:16`, `userland/capsule_driver_e1000/Capsule.mk:19`, `userland/capsule_driver_e1000/src/protocol/ops.rs:23` |
| `driver.rtl8139_0` | `service:4212:driver.rtl8139_0` | `0x1D8019` | healthcheck, link status, MAC address, TX packet, RX packet, stats | `userland/capsule_driver_rtl8139/src/main.rs:35` | [networking](../subsystems/networking/README.md) | `userland/capsule_driver_rtl8139/Capsule.mk:13`, `userland/capsule_driver_rtl8139/Capsule.mk:16`, `userland/capsule_driver_rtl8139/src/protocol/ops.rs:17` |
| `driver.rtl8169_0` | `service:4214:driver.rtl8169_0` | `0xF8019` | healthcheck, link status, MAC address, TX packet, RX packet, stats | `userland/capsule_driver_rtl8169/src/main.rs:40` | [networking](../subsystems/networking/README.md) | `userland/capsule_driver_rtl8169/Capsule.mk:13`, `userland/capsule_driver_rtl8169/Capsule.mk:16`, `userland/capsule_driver_rtl8169/src/protocol/ops.rs:17` |
| `driver.ahci0` | `service:4216:driver.ahci0` | `0xf8019` | healthcheck, controller info, port list | `userland/capsule_driver_ahci/src/main.rs:37` | [driver-ahci](driver-ahci/README.md) | `userland/capsule_driver_ahci/Capsule.mk:14`, `userland/capsule_driver_ahci/Capsule.mk:16`, `userland/capsule_driver_ahci/src/protocol/ops.rs:17` |
| `driver.hda0` | `service:4218:driver.hda0` | `0xF8119` | healthcheck, controller info, codec mask, stream layout, codec list, play tone, write PCM, stream start, stream stop | `userland/capsule_driver_hda/src/main.rs:37` | [driver-hda](driver-hda/README.md) | `userland/capsule_driver_hda/Capsule.mk:14`, `userland/capsule_driver_hda/Capsule.mk:17`, `userland/capsule_driver_hda/src/protocol/ops.rs:17` |
| `driver.nvme0` | `service:4220:driver.nvme0` | `0xF8019` | healthcheck, controller info, identify controller, identify namespace, SMART health | `userland/capsule_driver_nvme/src/main.rs:39` | [driver-nvme](driver-nvme/README.md) | `userland/capsule_driver_nvme/Capsule.mk:14`, `userland/capsule_driver_nvme/Capsule.mk:16`, `userland/capsule_driver_nvme/src/protocol/ops.rs:17` |
| `driver.usb_hid0` | `service:4222:driver.usb_hid0` | `0x200019` | healthcheck, probe config, feed keyboard report, feed mouse report, poll keys, poll mouse, get state | `userland/capsule_driver_usb_hid/src/main.rs:33` | [driver-usb-hid](driver-usb-hid/README.md) | `userland/capsule_driver_usb_hid/Capsule.mk:13`, `userland/capsule_driver_usb_hid/Capsule.mk:15`, `userland/capsule_driver_usb_hid/src/protocol/ops.rs:17` |
| `driver.usb_msc0` | `service:4224:driver.usb_msc0` | `0x19` | healthcheck, probe config, build inquiry, build read capacity, build read10, build write10, accept CSW, get state | `userland/capsule_driver_usb_msc/src/main.rs:32` | [driver-usb-msc](driver-usb-msc/README.md) | `userland/capsule_driver_usb_msc/Capsule.mk:13`, `userland/capsule_driver_usb_msc/Capsule.mk:18`, `userland/capsule_driver_usb_msc/src/protocol/ops.rs:17` |
| `driver.iwlwifi0` | `service:4228:driver.iwlwifi0` | `0xF8019` | healthcheck, device info, firmware info, RF state, DMA state, firmware stage, alive wait | `userland/capsule_driver_iwlwifi/src/main.rs:35` | [networking](../subsystems/networking/README.md) | `userland/capsule_driver_iwlwifi/Capsule.mk:12`, `userland/capsule_driver_iwlwifi/Capsule.mk:15`, `userland/capsule_driver_iwlwifi/src/protocol/ops.rs:9` |
| `driver.i2c_pci0` | `service:4230:driver.i2c_pci0` | `0x78019` | healthcheck, controller info, register snapshot, timing info, transfer, probe | `userland/capsule_driver_i2c_pci/src/main.rs:19` | [driver-i2c-pci](driver-i2c-pci/README.md) | `userland/capsule_driver_i2c_pci/Capsule.mk:13`, `userland/capsule_driver_i2c_pci/Capsule.mk:16`, `userland/capsule_driver_i2c_pci/src/protocol/ops.rs:1` |
| `driver.i2c_hid0` | `service:4232:driver.i2c_hid0` | `0x200119` | healthcheck, probe, descriptor | `userland/capsule_driver_i2c_hid/src/main.rs:32` | [driver-i2c-hid](driver-i2c-hid/README.md) | `userland/capsule_driver_i2c_hid/Capsule.mk:12`, `userland/capsule_driver_i2c_hid/Capsule.mk:14`, `userland/capsule_driver_i2c_hid/src/protocol/ops.rs:1` |

The `i2c_hid` capsule on this branch is the relative-mouse driver: it posts
relative pointer, wheel, and button events
(`userland/capsule_driver_i2c_hid/src/input/publish.rs:28`,
`userland/capsule_driver_i2c_hid/src/input/publish.rs:31`,
`userland/capsule_driver_i2c_hid/src/input/publish.rs:38`). The full Precision
Touchpad path with absolute coordinates lives on a separate branch and is not in
this tree.

There is a fourteenth display capsule in the source tree,
[driver-bga](driver-bga/README.md), which is not in the table above
because it is parked. `capsule_driver_bga` has source but no `Capsule.mk`, so it
has no service handle, no port, no capability mask, and no entry in the
build-and-sign system; its README calls it a parked source inventory for a
future brokered BGA display capsule
(`userland/capsule_driver_bga/README.md:5`). Treat it as reference source, not a
shipping driver.

## 2a. Maturity and hardware coverage

The contract table says what each driver exposes; it does not say how far each one is actually proven,
and the honest answer is uneven. This matrix is the load-bearing status. Coverage is split into QEMU
(the virtual model each driver was written against) and real hardware (silicon it has actually run on).
The maturity tag reads: PROVEN (a machine-checked proof crate or a live boot path that depends on it),
DEMONSTRATED (shown running but without a proof artifact), IMPLEMENTED (code complete, end-to-end result
not asserted in-repo), PARTIAL (only part of the device's function), STUB (framing or enumeration only).
Be brutal about the difference between "the code exists" and "a byte moved."

| Driver | Class | What works | Stub / partial / unproven | QEMU | Real HW | Tag |
|--------|-------|-----------|---------------------------|------|---------|-----|
| `virtio_gpu` (2D) | display | primary B8G8R8A8 surface, `SET_SCANOUT`/`TRANSFER_TO_HOST_2D`/`RESOURCE_FLUSH`; drives the desktop every frame | panel size from `GET_DISPLAY_INFO`, 1280x720 fallback; no EDID | yes, drives desktop | not on GPU silicon (compositor uses GOP there) | DEMONSTRATED |
| `virtio_gpu` (3D) | display | full VirGL/Gallium stream builder, boot `SUBMIT_3D` self-test, `virgl_ready` via `QUERY_CAPS` | no shipping capsule issues `SUBMIT_3D`; needs host virglrenderer behind `virtio-vga-gl` | probe only | no | IMPLEMENTED, NOT USED |
| `bga` | display | one-shot 1024x768x32 Bochs/BGA mode-set | parked: no `Capsule.mk`, no service, no broker path | Bochs/QEMU/VBox model | no | PARTIAL (parked) |
| `virtio_blk` | block | real split-virtqueue DMA read and write; serves `/data` at boot | write op gated to pid 0; no proof crate | yes, boot path | paravirtual only | DEMONSTRATED |
| `nvme` | block | admin + I/O queues, identify, SMART, 512 and 4096 LBA read/write engine | service exposes enumeration ops; read/write via block layer; no proof crate | yes | NVMe is real-HW class per coverage notes | IMPLEMENTED |
| `ahci` | block | ATA `READ_DMA_EXT`/`WRITE_DMA_EXT` engine, port enumeration | service exposes controller-info/port-list; polled, no IRQ path; no proof crate | yes | AHCI is real-HW class per coverage notes | IMPLEMENTED |
| `ps2_input` | input | 8042 keyboard + mouse, PIO + IRQ, posts to input ring | none material | yes | yes, real laptops | PROVEN |
| `usb_hid` | input | boot-protocol keyboard/mouse/tablet over xHCI, posts events | client of xHCI, which is weak on real silicon | yes | partial (xHCI-dependent) | DEMONSTRATED |
| `i2c_hid` | input | ACPI-described touchpad over `i2c_pci`, PTP report parse, posts events | depends on i2c_pci + correct ACPI `_CRS`; brittle on real HW | limited | partial | PARTIAL |
| `i2c_pci` | bus | Intel LPSS controller bring-up, SCL timing, transaction service | no DMA; real-HW SCL/clock quirks; enumeration + transfer only | limited | partial | PARTIAL |
| `xhci` | USB host | controller reset, port enumeration, slot/address, control + interrupt transfers | no bulk-transfer op, so mass-storage data path is unwired end to end | yes | partial | PARTIAL |
| `usb_msc` | block-over-USB | builds/validates BOT CBW/CSW and SCSI CDBs | runs no transfer, publishes no block device, holds zero broker grants | n/a (no I/O) | n/a | STUB |
| `hda` | audio | controller reset, codec probe, CORB/RIRB verbs, BDL, stream setup/run, PCM/tone ops | no proof or test that a sample reaches a speaker; playback unasserted | untested audio | untested audio | IMPLEMENTED, NOT PROVEN |
| `virtio_net` | network | full split-virtqueue tx/rx; best-covered NIC | none material; QEMU-only device | yes, 8 proof bins | paravirtual only | PROVEN |
| `e1000` | network | DMA descriptor rings, tx + rx, stats; MAC via Crypto-gated draw | real e1000e variants listed but not asserted | yes (`0x100E`), 6 proof bins | plausible, not asserted | DEMONSTRATED |
| `rtl8139` | network | PIO tx/rx, ring buffers | QEMU model device | yes, 7 proof bins | not asserted | DEMONSTRATED |
| `rtl8169` | network | MMIO tx/rx, descriptor rings | no proof crate; least-covered wired NIC | yes | not asserted | IMPLEMENTED |
| `rtl8821ce` | wifi | firmware download, association MLME, WPA2 four-way on real silicon (polled, no IRQ) | polled RX; live DHCP-over-this-driver proven off-tree (HP), not in launchpad CI | model-limited | yes, real silicon | DEMONSTRATED (real HW) |
| `virtio_rng` | entropy | virtqueue random fill | QEMU-only device | yes | paravirtual only | IMPLEMENTED |
| `iwlwifi` | wifi | firmware image parse and DMA transfer staging, register bring-up | ALIVE / NVM / PHY-cal / secure-boot not implemented; blocked on a `.pnvm` blob; association host-tested only | partial | not proven | IMPLEMENTED, NOT PROVEN |

Two cross-cutting honesty notes. First, the IOMMU is enforced under QEMU but has never been exercised on
real hardware, so every DMA-capable driver above rests on the software bounds of the
[DMA broker](../subsystems/hardware-broker/dma.md) plus the assumption of non-malicious device silicon,
not on hardware isolation. Second, the block and audio drivers whose services expose only enumeration
(`ahci`, `nvme` identify/SMART, `hda`) do have real data-path engines in-tree; what is unproven is the
end-to-end result, not the presence of the code.

## 3. Server form

NIC drivers run mutable driver state through their server loops after device
construction (`userland/capsule_driver_e1000/src/main.rs:38`,
`userland/capsule_driver_rtl8139/src/main.rs:35`,
`userland/capsule_driver_rtl8169/src/main.rs:40`,
`userland/capsule_driver_virtio_net/src/main.rs:36`). Storage-class drivers
enter server loops after driver setup or service bootstrap
(`userland/capsule_driver_virtio_blk/src/main.rs:30`,
`userland/capsule_driver_ahci/src/main.rs:37`,
`userland/capsule_driver_nvme/src/main.rs:39`,
`userland/capsule_driver_usb_msc/src/main.rs:32`). Input drivers expose PS/2,
USB HID, and I2C HID event/configuration protocols
(`userland/capsule_driver_ps2_input/src/protocol/ops.rs:16`,
`userland/capsule_driver_usb_hid/src/protocol/ops.rs:17`,
`userland/capsule_driver_i2c_hid/src/protocol/ops.rs:1`).

## 4. Input driver event path

Input drivers do not send GUI events directly to apps. They normalize hardware
events into `InputEvent` and post them into the kernel input ring. The
`input_router` capsule drains that ring in bounded batches, applies grabs,
routes pointer events through WM hit testing, routes key events through WM focus,
and delivers `NINP` frames to subscribers
(`userland/capsule_input_router/src/sources/kernel_ring.rs:17`,
`userland/capsule_input_router/src/sources/kernel_ring.rs:25`,
`userland/capsule_input_router/src/sources/kernel_ring.rs:27`,
`userland/capsule_input_router/src/server/runner.rs:30`,
`userland/capsule_input_router/src/server/runner.rs:43`,
`userland/capsule_input_router/src/server/runner.rs:44`,
`userland/capsule_input_router/src/server/runner.rs:49`).

```
+----------------+
| PS/2 driver    |
| USB HID driver |
| I2C HID driver |
+-------+--------+
        |
+-------+--------+
| mk_input_event |
| post           |
+-------+--------+
        |
+-------+--------+
| kernel ring    |
+-------+--------+
        |
+-------+--------+
| input_router   |
| drain batch    |
+-------+--------+
        |
+-------+--------+
| WM focus and   |
| topmost query  |
+-------+--------+
        |
+-------+--------+
| NINP delivery  |
| app or shell   |
+----------------+
```

The router dispatches grabbed event kinds to the grab holder before normal
routing. Pointer kinds are routed through the pointer path, keyboard kinds
through the keyboard path, and other subscribed kinds are fanned out by
subscription match
(`userland/capsule_input_router/src/route/dispatch.rs:28`,
`userland/capsule_input_router/src/route/dispatch.rs:29`,
`userland/capsule_input_router/src/route/dispatch.rs:37`,
`userland/capsule_input_router/src/route/dispatch.rs:40`,
`userland/capsule_input_router/src/route/dispatch.rs:43`,
`userland/capsule_input_router/src/route/dispatch.rs:61`,
`userland/capsule_input_router/src/route/dispatch.rs:65`). Keyboard routing asks
WM for focus and falls back to the shell pid when WM has no focused owner
(`userland/capsule_input_router/src/route/keyboard.rs:25`,
`userland/capsule_input_router/src/route/keyboard.rs:27`). Pointer routing
refreshes display bounds, applies cursor state, mirrors pointer events to the
shell, queries the topmost target, and routes to the shell or target window
(`userland/capsule_input_router/src/route/pointer/route_pointer.rs:28`,
`userland/capsule_input_router/src/route/pointer/route_pointer.rs:29`,
`userland/capsule_input_router/src/route/pointer/route_pointer.rs:30`,
`userland/capsule_input_router/src/route/pointer/route_pointer.rs:31`,
`userland/capsule_input_router/src/route/pointer/route_pointer.rs:32`,
`userland/capsule_input_router/src/route/pointer/route_pointer.rs:35`). Delivery
encodes the event into the fixed input envelope and sends it to the target pid
(`userland/capsule_input_router/src/route/deliver.rs:24`,
`userland/capsule_input_router/src/route/deliver.rs:28`,
`userland/capsule_input_router/src/route/deliver.rs:29`,
`userland/capsule_input_router/src/route/deliver.rs:30`).

The producer table is written left to right: hardware work, normalized event
kinds, and the exact post or poll point. That is the chain to inspect when a key
or mouse event is missing from the desktop.

```
+--------------------------+
| ps2 pump                 |
+------------+-------------+
             |
+------------+-------------+
| usb hid poll             |
+------------+-------------+
             |
+------------+-------------+
| i2c hid poll             |
+------------+-------------+
             |
+------------+-------------+
| normalized InputEvent    |
+------------+-------------+
             |
+------------+-------------+
| kernel input ring        |
+--------------------------+
```

| Producer | Hardware path | Posted event kinds | Source |
|----------|---------------|--------------------|--------|
| `driver.ps2_kbd0` | Startup retries setup until a driver object is returned, then enters the server loop. Each loop pumps IRQ sequence state, drains PS/2 data, acknowledges keyboard and auxiliary IRQ grants, then services IPC. | Keyboard translation posts key-down and key-up. Mouse publishing posts relative pointer movement, wheel, button-down, and button-up. | Startup at `userland/capsule_driver_ps2_input/src/main.rs:31` to `userland/capsule_driver_ps2_input/src/main.rs:45`, pump at `userland/capsule_driver_ps2_input/src/server/pump.rs:24` to `userland/capsule_driver_ps2_input/src/server/pump.rs:45`, server loop at `userland/capsule_driver_ps2_input/src/server/runner.rs:32` to `userland/capsule_driver_ps2_input/src/server/runner.rs:73`, keyboard post at `userland/capsule_driver_ps2_input/src/keymap/post.rs:18` to `userland/capsule_driver_ps2_input/src/keymap/post.rs:31`, mouse post at `userland/capsule_driver_ps2_input/src/mouse/post.rs:24` to `userland/capsule_driver_ps2_input/src/mouse/post.rs:52`. |
| `driver.usb_hid0` | Startup initializes heap, waits until the xHCI service can be resolved, enumerates connected xHCI ports, then polls HID endpoints and rescans when no endpoints are present for the rescan interval. | Keyboard publishing posts key-down and key-up with modifier flags and mapped special keys. Mouse publishing posts relative pointer movement, wheel, button-down, and button-up. | Startup at `userland/capsule_driver_usb_hid/src/main.rs:32` to `userland/capsule_driver_usb_hid/src/main.rs:37`, xHCI lookup at `userland/capsule_driver_usb_hid/src/orchestrator/run.rs:19` to `userland/capsule_driver_usb_hid/src/orchestrator/run.rs:28`, enumeration at `userland/capsule_driver_usb_hid/src/orchestrator/enumerate/run.rs:25` to `userland/capsule_driver_usb_hid/src/orchestrator/enumerate/run.rs:36`, poll loop at `userland/capsule_driver_usb_hid/src/orchestrator/poll/run.rs:26` to `userland/capsule_driver_usb_hid/src/orchestrator/poll/run.rs:44`, keyboard post at `userland/capsule_driver_usb_hid/src/hid/post_key.rs:32` to `userland/capsule_driver_usb_hid/src/hid/post_key.rs:66`, mouse post at `userland/capsule_driver_usb_hid/src/hid/post_mouse.rs:24` to `userland/capsule_driver_usb_hid/src/hid/post_mouse.rs:49`, shared post at `userland/capsule_driver_usb_hid/src/hid/post_wire.rs:19` to `userland/capsule_driver_usb_hid/src/hid/post_wire.rs:22`. |
| `driver.i2c_hid0` | Startup resolves the I2C controller, probes the bus for a HID descriptor, records address, descriptor length, input register, and input report length, then the server loop polls input after every receive timeout. | Parsed mouse samples post relative pointer movement, wheel, button-down, and button-up. | Startup at `userland/capsule_driver_i2c_hid/src/main.rs:31` to `userland/capsule_driver_i2c_hid/src/main.rs:39`, setup at `userland/capsule_driver_i2c_hid/src/setup.rs:5` to `userland/capsule_driver_i2c_hid/src/setup.rs:19`, server poll point at `userland/capsule_driver_i2c_hid/src/server/runner.rs:15` to `userland/capsule_driver_i2c_hid/src/server/runner.rs:33`, I2C read and report parse at `userland/capsule_driver_i2c_hid/src/input/poll.rs:22` to `userland/capsule_driver_i2c_hid/src/input/poll.rs:34`, event publishing at `userland/capsule_driver_i2c_hid/src/input/publish.rs:25` to `userland/capsule_driver_i2c_hid/src/input/publish.rs:45`, shared post at `userland/capsule_driver_i2c_hid/src/input/post.rs:19` to `userland/capsule_driver_i2c_hid/src/input/post.rs:30`. |

## 6. Security analysis

Every one of the eighteen drivers is a ring-3 capsule, and not one of them runs in the kernel. The
kernel owns no keyboard controller, no NIC ring, no NVMe queue; it owns the [hardware
broker](../subsystems/hardware-broker/README.md), the input ring, and the capability check, and it lends
the hardware to a capsule that asks correctly. A driver reaches its device through four broker grants,
each a separate call checked against the same claim epoch and each revoked when the capsule exits:
[claim](../subsystems/hardware-broker/claim.md) takes exclusive ownership of the device and returns the
epoch, [mmio](../subsystems/hardware-broker/mmio.md) maps a slice of a BAR into the capsule's address
space, [irq](../subsystems/hardware-broker/irq.md) binds the device line to a kernel-delivered
notification, and [dma](../subsystems/hardware-broker/dma.md) hands back a DMA-coherent buffer, with a
fifth path, [pio](../subsystems/hardware-broker/pio.md), minting a kernel-mediated `in`/`out` window
against a port BAR. A driver holds exactly the subset of these its device needs and nothing more.

The mask is where that least-privilege split is written down, and it decodes exactly to the grants a
device requires (bits from `src/capabilities/types/bit.rs`; the enum is in `src/capabilities/types/defs.rs`).
The masks are not interchangeable; each is the smallest set for its bus. Two bits beyond the broker
quartet are worth calling out: `Debug` (`0x100`) is the serial-log grant most driver manifests now carry,
and `Crypto` (`0x20`) is held only by the NICs whose MAC address is drawn through a Crypto-gated syscall
(`e1000`, `rtl8821ce`):

| Mask | Capabilities beyond CoreExec/IPC/Memory | Drivers |
|------|-----------------------------------------|---------|
| `0x19` | none | `usb_msc` |
| `0x200019` | InputSource | `usb_hid` |
| `0x200119` | InputSource, Debug | `i2c_hid` |
| `0x78019` | DeviceEnum, Driver, Mmio, Irq | `i2c_pci` |
| `0xf8019` | DeviceEnum, Driver, Mmio, Irq, Dma | `ahci`, `xhci`, `nvme`, `rtl8169`, `iwlwifi` |
| `0xF8039` | DeviceEnum, Driver, Mmio, Irq, Dma, Crypto | `e1000` |
| `0xF8119` | DeviceEnum, Driver, Mmio, Irq, Dma, Debug | `hda` |
| `0xF8139` | DeviceEnum, Driver, Mmio, Irq, Dma, Crypto, Debug | `rtl8821ce` |
| `0x1D8019` | DeviceEnum, Driver, Irq, Dma, Pio | `rtl8139` |
| `0x1F8019` | DeviceEnum, Driver, Mmio, Irq, Dma, Pio | `virtio_rng` |
| `0x1F8119` | DeviceEnum, Driver, Mmio, Irq, Dma, Pio, Debug | `virtio_blk`, `virtio_net` |
| `0x1F9119` | DeviceEnum, Driver, Mmio, Irq, Dma, Pio, GraphicsSurfaceCreate, Debug | `virtio_gpu` |
| `0x358019` | DeviceEnum, Driver, Irq, Pio, InputSource | `ps2_kbd` |

Several properties are worth reading straight off that table. The HID drivers, `usb_hid` and `i2c_hid`,
hold `InputSource` and nothing else in the hardware column: they carry no Mmio, no Irq, no Dma, no Pio,
because they reach their hardware through another capsule (the xHCI driver and the i2c bus driver
respectively) over IPC. A compromised HID report parser, which is the most complex and most exposed code
in the input path, cannot map a register, take an interrupt, or program DMA; the worst it can do is post
forged input events, which are bounded by the ring. The inverse is `i2c_pci`: it has Mmio and Irq but
*not* InputSource, so the capsule that drives the controller registers cannot inject a keystroke. The
right to touch the hardware and the right to produce input are held by different capsules on purpose,
which is the [input drivers](../subsystems/input/drivers.md) page in one sentence. `usb_msc` is the
extreme case, mask `0x19`, the three baseline bits and no hardware capability at all, because it builds
SCSI command blocks and hands them to the xHCI driver rather than touching a controller itself.

`Dma` is the capability to watch, and the split between `0x78019` and `0xf8019` is exactly the line
between devices that bus-master and devices that do not. `hda` and `i2c_pci` get Mmio and Irq but no
Dma; AHCI, the NICs, NVMe, xHCI, and iwlwifi get Dma because they move data through descriptor rings in
RAM. The [broker's DMA grant](../subsystems/hardware-broker/dma.md) bounds what a *capsule* may allocate
and program (a per-class page ceiling, a zero-scrub before the frames leave the kernel, and an epoch
check against use-after-release), but it is honest about the one bound it does not enforce: the IOMMU
backend is behind the `nonos-arch-iommu` feature and is not engaged in the shipping builds, so the
address the broker returns is a raw physical address and a device programmed by a driver can in principle
DMA to any physical page regardless of the grant. DMA safety therefore rests on the software bounds plus
the assumption of non-malicious device hardware, and this is the same boundary every DMA-capable driver
in the table inherits. The storage drivers reach the [block device](../subsystems/storage/block-device.md)
layer and ultimately the [vfs capsule](../subsystems/storage/vfs-capsule.md) above them, none of which
gains the driver any authority the mask did not already grant.

## 7. Debugging

The drivers are instrumented so that a machine which boots with a dead device names the stage that
failed rather than going silent. The debugging follows the spawn, then the grant, then the device, in
that order.

The first question is whether the driver capsule loaded. Every driver is spawned through
`super::boot::capsule(prefix, ...)` in its spawn-plan group
(`src/userspace/init/spawn_plan/drivers_storage.rs`, `drivers_nic.rs`, `drivers_usb.rs`,
`drivers_bus.rs`, `drivers_input.rs`, `drivers_virtio.rs`), which routes to `capsule_boot::boot`
(`src/userspace/init/capsule_boot/run.rs:29`) and prints `boot_log::ok(prefix, "capsule spawned")` on
success or `boot_log::error(...)` on failure. So a live driver prints a line naming it: `[DRIVER-AHCI]`,
`[DRIVER-NVME]`, `[DRIVER-HDA]`, `[DRIVER-PS2-INPUT]`, and the rest, each followed by `capsule spawned`.
If that line is absent the capsule never ran: its ELF failed signature verification or its manifest asked
for a capability outside policy, and the spawn was refused before any driver code executed. This is the
same marker the [input drivers](../subsystems/input/drivers.md) page uses to answer "did the driver even
load."

If the driver spawned but the device is dead, the next stage is the broker grant, and the broker prints
its own markers as it hands hardware over. An MMIO map walks through `[MMIO] claim`, `[MMIO] device`,
`[MMIO] reserve`, `[MMIO] va`, `[MMIO] map`, and `[MMIO] record`
(`src/hardware/broker/mmio/map.rs`), so a driver that spawned but never reached `[MMIO] record` for its
device failed partway through the map. A DMA grant prints `[DMA]` lines and names the exact failure class
when it refuses (`src/hardware/broker/dma/map/mod.rs`): `[DMA] validate not-claimed` means the driver
asked for DMA on a device it never claimed, `[DMA] validate stale-epoch` means its claim lapsed and was
re-taken, `[DMA] validate bad-length-class` means the request exceeded the per-class ceiling, and
`[DMA] alloc no-memory` means the physical frames were not available. These distinguish a driver bug
(asking wrong) from a resource problem (nothing free) without reading the driver's code. The claim
itself is the gate before any of these: a claim is refused with `AlreadyClaimed`
(`src/hardware/broker/claim.rs:51`) when another capsule already holds the device, so two drivers
contending for one device shows up as the second one never getting past claim.

If the grant succeeded but nothing happens, the failure has moved to the device or its interrupt, and
the distinction is between enumeration and drive. Whether the firmware exposed the device at all is a
broker-table question, separate from whether a driver spawned: a device absent from the broker's device
table is a firmware or ACPI problem, not a driver defect, and the device-census tooling described on the
[input drivers](../subsystems/input/drivers.md) page renders that table so the two can be told apart
before any driver is blamed. An enumerated device with a spawned driver that still produces nothing is
usually the interrupt: the [irq grant](../subsystems/hardware-broker/irq.md) bound the line but the line
never fires, which on real hardware is typically GSI-to-vector routing through the IOAPIC rather than a
driver fault. For the input drivers specifically, the producer table in section 4 above lists the exact
poll and post points to inspect when a key or mouse event goes missing, and the
[input drivers](../subsystems/input/drivers.md) page covers isolating the kernel ring from the driver
with a synthetic-event probe.

# Driver Capsules

This page documents the user-mode hardware driver capsules. Read
[Capsule Inventory](capsules.md), [Hardware Broker](../subsystems/hardware-broker.md),
[Input](../subsystems/input.md), [Graphics](../subsystems/graphics.md), and
[Storage](../subsystems/storage.md) first.

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

| Capsule | Service | Caps | Protocol operations | Entrypoint | Spec refs |
|---------|---------|------|---------------------|------------|-----------|
| `driver.virtio_rng` | `service:4200:driver.virtio_rng` | `0x1F8019` | fill random, healthcheck | `userland/capsule_driver_virtio_rng/src/main.rs:35` | `userland/capsule_driver_virtio_rng/Capsule.mk:13`, `userland/capsule_driver_virtio_rng/Capsule.mk:17`, `userland/capsule_driver_virtio_rng/src/protocol/ops.rs:21` |
| `driver.virtio_blk0` | `service:4202:driver.virtio_blk0` | `0x1F8019` | healthcheck, capacity, read blocks, write blocks, flush | `userland/capsule_driver_virtio_blk/src/main.rs:30` | `userland/capsule_driver_virtio_blk/Capsule.mk:13`, `userland/capsule_driver_virtio_blk/Capsule.mk:16`, `userland/capsule_driver_virtio_blk/src/protocol/ops.rs:16` |
| `driver.virtio_net0` | `service:4204:driver.virtio_net0` | `0x1F8019` | healthcheck, link status, MAC address, TX packet, RX packet | `userland/capsule_driver_virtio_net/src/main.rs:36` | `userland/capsule_driver_virtio_net/Capsule.mk:14`, `userland/capsule_driver_virtio_net/Capsule.mk:17`, `userland/capsule_driver_virtio_net/src/protocol/ops.rs:21` |
| `driver.virtio_gpu0` | `service:4226:driver.virtio_gpu0` | `0x1F9019` | healthcheck, controller info, display info, controlq state, query caps, create resource, attach backing, transfer to host, set scanout, flush, mode list, primary surface | `userland/capsule_driver_virtio_gpu/src/main.rs:35` | `userland/capsule_driver_virtio_gpu/Capsule.mk:12`, `userland/capsule_driver_virtio_gpu/Capsule.mk:16`, `userland/capsule_driver_virtio_gpu/src/protocol/ops.rs:16` |
| `driver.xhci0` | `service:4206:driver.xhci0` | `0xF8019` | healthcheck, controller status, port status, enable slot, disable slot, address device, device descriptor, config descriptor, transfer ring allocation, control transfer, interrupt in | `userland/capsule_driver_xhci/src/main.rs:36` | `userland/capsule_driver_xhci/Capsule.mk:13`, `userland/capsule_driver_xhci/Capsule.mk:16`, `userland/capsule_driver_xhci/src/protocol/ops.rs:16` |
| `driver.ps2_kbd0` | `service:4208:driver.ps2_kbd0` | `0x358019` | healthcheck, poll events, get state, controller status, poll mouse | `userland/capsule_driver_ps2_input/src/main.rs:31` | `userland/capsule_driver_ps2_input/Capsule.mk:13`, `userland/capsule_driver_ps2_input/Capsule.mk:17`, `userland/capsule_driver_ps2_input/src/protocol/ops.rs:16` |
| `driver.e1000_0` | `service:4210:driver.e1000_0` | `0xF8019` | healthcheck, link status, MAC address, TX packet, RX packet, stats | `userland/capsule_driver_e1000/src/main.rs:38` | `userland/capsule_driver_e1000/Capsule.mk:16`, `userland/capsule_driver_e1000/Capsule.mk:19`, `userland/capsule_driver_e1000/src/protocol/ops.rs:23` |
| `driver.rtl8139_0` | `service:4212:driver.rtl8139_0` | `0x1D8019` | healthcheck, link status, MAC address, TX packet, RX packet, stats | `userland/capsule_driver_rtl8139/src/main.rs:35` | `userland/capsule_driver_rtl8139/Capsule.mk:13`, `userland/capsule_driver_rtl8139/Capsule.mk:16`, `userland/capsule_driver_rtl8139/src/protocol/ops.rs:17` |
| `driver.rtl8169_0` | `service:4214:driver.rtl8169_0` | `0xF8019` | healthcheck, link status, MAC address, TX packet, RX packet, stats | `userland/capsule_driver_rtl8169/src/main.rs:40` | `userland/capsule_driver_rtl8169/Capsule.mk:13`, `userland/capsule_driver_rtl8169/Capsule.mk:16`, `userland/capsule_driver_rtl8169/src/protocol/ops.rs:17` |
| `driver.ahci0` | `service:4216:driver.ahci0` | `0x78019` | healthcheck, controller info, port list | `userland/capsule_driver_ahci/src/main.rs:37` | `userland/capsule_driver_ahci/Capsule.mk:14`, `userland/capsule_driver_ahci/Capsule.mk:17`, `userland/capsule_driver_ahci/src/protocol/ops.rs:17` |
| `driver.hda0` | `service:4218:driver.hda0` | `0x78019` | healthcheck, controller info, codec mask, stream layout, codec list | `userland/capsule_driver_hda/src/main.rs:37` | `userland/capsule_driver_hda/Capsule.mk:14`, `userland/capsule_driver_hda/Capsule.mk:17`, `userland/capsule_driver_hda/src/protocol/ops.rs:17` |
| `driver.nvme0` | `service:4220:driver.nvme0` | `0xF8019` | healthcheck, controller info, identify controller, identify namespace, SMART health | `userland/capsule_driver_nvme/src/main.rs:39` | `userland/capsule_driver_nvme/Capsule.mk:14`, `userland/capsule_driver_nvme/Capsule.mk:17`, `userland/capsule_driver_nvme/src/protocol/ops.rs:17` |
| `driver.usb_hid0` | `service:4222:driver.usb_hid0` | `0x200019` | healthcheck, probe config, feed keyboard report, feed mouse report, poll keys, poll mouse, get state | `userland/capsule_driver_usb_hid/src/main.rs:33` | `userland/capsule_driver_usb_hid/Capsule.mk:13`, `userland/capsule_driver_usb_hid/Capsule.mk:15`, `userland/capsule_driver_usb_hid/src/protocol/ops.rs:17` |
| `driver.usb_msc0` | `service:4224:driver.usb_msc0` | `0x19` | healthcheck, probe config, build inquiry, build read capacity, build read10, build write10, accept CSW, get state | `userland/capsule_driver_usb_msc/src/main.rs:32` | `userland/capsule_driver_usb_msc/Capsule.mk:13`, `userland/capsule_driver_usb_msc/Capsule.mk:18`, `userland/capsule_driver_usb_msc/src/protocol/ops.rs:17` |
| `driver.iwlwifi0` | `service:4228:driver.iwlwifi0` | `0xF8019` | healthcheck, device info, firmware info, RF state, DMA state, firmware stage, alive wait | `userland/capsule_driver_iwlwifi/src/main.rs:35` | `userland/capsule_driver_iwlwifi/Capsule.mk:12`, `userland/capsule_driver_iwlwifi/Capsule.mk:15`, `userland/capsule_driver_iwlwifi/src/protocol/ops.rs:9` |
| `driver.i2c_pci0` | `service:4230:driver.i2c_pci0` | `0x78019` | healthcheck, controller info, register snapshot, timing info, transfer, probe | `userland/capsule_driver_i2c_pci/src/main.rs:19` | `userland/capsule_driver_i2c_pci/Capsule.mk:13`, `userland/capsule_driver_i2c_pci/Capsule.mk:16`, `userland/capsule_driver_i2c_pci/src/protocol/ops.rs:1` |
| `driver.i2c_hid0` | `service:4232:driver.i2c_hid0` | `0x200019` | healthcheck, probe, descriptor | `userland/capsule_driver_i2c_hid/src/main.rs:32` | `userland/capsule_driver_i2c_hid/Capsule.mk:12`, `userland/capsule_driver_i2c_hid/Capsule.mk:14`, `userland/capsule_driver_i2c_hid/src/protocol/ops.rs:1` |

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

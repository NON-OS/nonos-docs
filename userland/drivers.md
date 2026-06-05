# Driver Capsules

This page documents the user-mode hardware driver capsules. Read
[Capsule Inventory](capsules.md), [Hardware Broker](../subsystems/hardware-broker.md),
[Input](../subsystems/input.md), [Graphics](../subsystems/graphics.md), and
[Storage](../subsystems/storage.md) first.

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
  | virtio         |
  +-------+--------+
          |
  +-------+--------+
  | bus input nic  |
  +-------+--------+
          |
  +-------+--------+
  | usb storage    |
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


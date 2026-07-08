# The Device Table and PCI Config

Before anything can be claimed, the broker has to know what devices exist. It builds a table from
PCI enumeration (plus registered platform devices), classifies each device so a capsule can
discover the kind of hardware it drives, and mediates the narrow set of PCI config-space writes a
driver legitimately needs. This page documents discovery, classification, and the config
allowlist. The code is under `src/hardware/broker/table/`, `class.rs`, and `pci/`.

## The device table

`init_from_pci` (`src/hardware/broker/table/init.rs:28`) turns the PCI enumeration into the
broker's device table and a parallel PCI handle index:

```
  init_from_pci(devices):
      for each PCI device, index idx:
          records.push(record_from_pci(idx, dev))     // device_id = idx
          handles.push(PciHandle { idx, address, bars, msix })
      TABLE = records
      pci_index::install(handles)
```

Each device gets a stable `device_id` (its enumeration index), a `DeviceRecord` with its BARs and
IRQ pin/line, and a `PciHandle` with its config address and MSI-X capability. The `DeviceRecord`
is what the [MMIO](mmio.md) and [IRQ](irq.md) paths resolve a request against; the `PciHandle` is
the kernel-private side table those paths use for capability walks and config access, which the
capsule never sees. Platform (non-PCI) devices are added through `register_platform_device`
(`init.rs:44`), which assigns a device id above the PCI range.

## Classification

`classify_pci` (`class.rs:49`) maps a device's PCI class, subclass, and prog-if to a broker class
id, and `MkDeviceList` surfaces those ids so a capsule can find the hardware it knows how to
drive:

```
  RNG 0x0001   BLOCK 0x0010   NETWORK 0x0020   DISPLAY 0x0030   INPUT 0x0040
  AUDIO 0x0050   SERIAL 0x0060   USB_HOST 0x0070   USB_HOST_XHCI 0x0071   OTHER 0xFFFF
```

Anything the broker does not specifically classify lands in `OTHER` so the table still surfaces
it rather than hiding it. The xHCI id is a deliberate subset of USB host: a controller advertising
the xHCI prog-if gets `USB_HOST_XHCI` while older UHCI/OHCI/EHCI controllers stay on the generic
`USB_HOST` id, so a userland xHCI driver matches on the specific id and never tries to drive a
controller it does not understand. The class id also selects the [DMA](dma.md) page ceiling.

## The PCI config allowlist

A driver sometimes needs to write device config space, most importantly to set the bus-master
enable bit before DMA. Rather than expose config space, the broker allows exactly two writes and
rejects everything else. The entire authority is one pure validator (`pci/allowlist.rs:33`):

```
  validate(req, msix, current):
      if offset == Command:        only Bus Master Enable (bit 2) may flip
      if offset == MSI-X Control:  only Function Mask + MSI-X Enable may flip
      else:                        OffsetNotAllowed
```

Every other config write, BAR programming, interrupt line, device and vendor IDs, status,
expansion ROM, capability-pointer mutation, PCIe and AER, is rejected before it reaches the bus.
`MkPciConfigWrite` (`pci/write.rs:26`) resolves the caller's ownership, reads the current register,
runs the validator, and applies only the allowed action, so a capsule cannot reprogram a BAR to
point its device somewhere else or rewrite a field the kernel relies on. The bus-master bit is
allowed because a DMA-capable driver genuinely needs it; the MSI-X control bits are allowed
because the driver enables and masks its own interrupts, but the table entries themselves are
programmed only by the kernel on the [IRQ](irq.md) bind path.

## Source

```
  src/hardware/broker/table/init.rs        init_from_pci, register_platform_device
  src/hardware/broker/table/pci_record.rs  record_from_pci, DeviceRecord construction
  src/hardware/broker/class.rs             classify_pci and the class ids
  src/hardware/broker/pci/allowlist.rs     the two-write config allowlist
  src/hardware/broker/pci/write.rs         MkPciConfigWrite orchestration
```

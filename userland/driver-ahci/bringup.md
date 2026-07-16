# AHCI bring-up and broker grants

This page mirrors `src/discover.rs`, `src/setup/`, `src/controller/`, and `src/handles/`: the one-shot
privileged path that turns a PCI enumeration entry into a `Driver` with an open register window, a bound
interrupt, and a brought-up SATA port. Every step here is a broker syscall or a register access on a
window the broker mapped. It runs exactly once, in `setup::run`, and unwinds prior grants on any failure.
For the request loop that runs against the resulting `Driver`, see [operations.md](operations.md); for the
command engine that `block_port::bring_up` drives, see [engine.md](engine.md).

## The one-shot sequence

`setup::run` is the whole privileged path in one function (`src/setup/sequence.rs:26`):

```
  find_ahci            -> device_id, irq_line, abar_size          discover.rs:34
  claim::claim         -> claim epoch                             setup/claim.rs:21
  pci::enable_bus_master  set the PCI bus-master bit              setup/pci.rs:21
  mmio::map            -> user_va, grant_id  (BAR5 / ABAR)        setup/mmio.rs:22
  irq::bind            -> irq grant_id                            setup/irq.rs:22
  BrokerHandles::new   take ownership of the three grants         handles/broker_handles.rs:26
  enable_ahci          set GHC.AE, clear HBA_IS                   controller/enable.rs:20
  ControllerInfo::read read CAP/GHC/PI/VS/CAP2, derive port count controller/info.rs:31
  scan_ports           snapshot every implemented port            controller/scan_ports.rs:26
  block_port::bring_up bring up the first present SATA port       setup/block_port.rs:22
```

The unwinding is explicit and ordered. If `enable_bus_master` fails, the device claim is released before
returning (`src/setup/sequence.rs:29`). `mmio::map` releases the device on its own failure
(`src/setup/mmio.rs:26`), and `irq::bind` unmaps the ABAR and releases the device on its failure
(`src/setup/irq.rs:26`). Once all three grants succeed they are handed to `BrokerHandles`, which owns them
for the life of the process and frees them in reverse on drop (below). A failure to bring up a port is
not fatal: `block_port::bring_up` returns `None` and the driver still serves controller-info and port-list
(`src/setup/block_port.rs:35`).

## Discovery

`find_ahci` asks the broker for the block-class device list with `mk_device_list`, capped at 32 records,
and returns the first candidate (`src/discover.rs:34`). `is_candidate` requires a PCI device
(`bus_kind == BUS_KIND_PCI`), block class, PCI class storage (0x01), subclass SATA (0x06), prog-IF AHCI
(0x01), a BAR count past BAR5, and BAR5 present as MMIO of at least 4 KiB (`src/discover.rs:52`,
`src/constants/pci.rs:18`).

The IRQ line is deliberately not part of the match. A controller that reports `irq_line = 0xff`, common on
APIC and MSI laptop platforms with no legacy PIC routing, is still a valid candidate, because the I/O
path polls the port completion registers and never waits on the interrupt. The comment records that
requiring a routed line here previously rejected working SATA controllers on real hardware
(`src/discover.rs:60`). The record it returns carries only the `device_id`, the `irq_line`, and BAR5's
size (`src/discover.rs:42`).

## Claim, bus master, MMIO, IRQ

Each of the four privileged grants is one broker syscall wrapped in a helper that maps a negative return
to `AhciError::BrokerCallFailed`.

- **Claim.** `claim::claim` calls `mk_device_claim(device_id)` and returns the claim epoch. That epoch is
  carried by every later broker call so the kernel can bind the grant to this claim
  (`src/setup/claim.rs:21`).
- **Bus master.** `enable_bus_master` calls `mk_pci_config_write` against the claimed function to set the
  bus-master bit in the PCI command register (`MK_PCI_CFG_COMMAND`, `MK_PCI_CMD_BUS_MASTER`), which the
  controller needs to drive DMA (`src/setup/pci.rs:21`). The config write goes through the broker, not
  raw config space.
- **Map ABAR.** `mmio::map` calls `mk_mmio_map` for BAR5, the AHCI register window, with the claim epoch
  and the discovered size, and returns a user VA, a length, and a grant id
  (`src/setup/mmio.rs:22`, `src/constants/pci.rs:18`). Only BAR5 is mapped; the driver never sees any
  other register space.
- **Bind IRQ.** `irq::bind` calls `mk_irq_bind` with the discovered `irq_line` and the claim epoch and
  returns a grant id and a vector (`src/setup/irq.rs:22`). The interrupt is only polled and acked at
  runtime; it is not required for command completion.

The [claim](../../subsystems/hardware-broker/claim.md), [mmio](../../subsystems/hardware-broker/mmio.md),
and [irq](../../subsystems/hardware-broker/irq.md) broker pages describe how each grant is validated,
bounded to the claim epoch, and revoked.

## The register wrapper

Once the ABAR is mapped, `Regs::new(user_va)` wraps the base address and every register access goes
through its volatile 32-bit `r32`/`w32` at `base + offset` (`src/regs/mmio.rs:29`, `src/setup/sequence.rs:38`).
`Regs` is `Copy` and holds only the base, so it is passed by value into the controller and engine code
that needs it. The HBA global and per-port register offsets it reads are the constants in
`src/constants/regs.rs`.

## Enable AHCI mode and read the controller

`enable_ahci` reads `HBA_GHC`, sets `GHC.AE` (bit 31), and clears the global interrupt status by writing
all-ones to `HBA_IS` (`src/controller/enable.rs:20`, `src/constants/regs.rs:17`). There is no
controller-wide `GHC.HR` HBA reset in this slice; the bring-up asserts AHCI mode and then does a per-port
engine stop/start cycle instead (see [engine.md](engine.md)).

`ControllerInfo::read` then reads `CAP`, `GHC`, `PI`, `VS`, and `CAP2`, and derives the port count from
`CAP` bits 0..4 plus one (`src/controller/info.rs:31`). This same struct is re-read live to answer
`OP_CONTROLLER_INFO` (see [operations.md](operations.md)).

## Port scan

`scan_ports` walks each bit set in `PI`, up to `MAX_PORTS` (32), and builds a `PortInfo` snapshot for each
implemented port (`src/controller/scan_ports.rs:26`, `src/constants/port.rs:17`). The port register base
is `PORT_BASE + index * PORT_STRIDE`, that is `0x100 + index * 0x80` (`src/constants/regs.rs:26`). For
each it reads `PxSSTS`, `PxSIG`, `PxIS`, `PxCMD`, `PxTFD`, `PxSERR`, `PxSACT`, and `PxCI`, and records the
index, an implemented flag, a present flag, and a kind byte.

A port is marked present only when `PxSSTS.DET == 3` and `PxSSTS.IPM` is 1 or 6, that is a device is
detected with phy communication and the interface is in an active or slumber power state
(`src/controller/scan_ports.rs:65`). A present port's kind is classified from its `PxSIG` signature:
`0x0000_0101` SATA, `0xeb14_0101` ATAPI, `0xc33c_0101` SEMB, `0x9669_0101` port multiplier, anything else
unknown (`src/controller/signature.rs:20`, `src/constants/regs.rs:57`). A non-present port is left with
kind none. This snapshot is what `OP_PORT_LIST` returns unchanged.

## Bring up the block port

`block_port::bring_up` walks the scanned ports and picks the first that is both present and SATA-signature,
calling `init_port` on it (`src/setup/block_port.rs:22`). `init_port` allocates the DMA regions, programs
the port, and issues `IDENTIFY`; that hardware path is the [engine](engine.md) page. If `init_port`
succeeds the port becomes `driver.block = Some(Port)`; if no port qualifies, `bring_up` returns `None` and
the block ops answer `E_NODEV`.

## The Driver and its grant owners

`setup::run` returns a `Driver` that owns everything acquired during bring-up: the `BrokerHandles`, the
`Regs` window, the `ControllerInfo`, the `PortInfo` array, and the optional block `Port`
(`src/setup/driver.rs:23`). All of the driver's broker grants are held in two owners that free them on
drop:

- `BrokerHandles` holds the device claim, the ABAR mapping, and the IRQ bind. Its `Drop` releases them in
  order: `mk_irq_unbind`, then `mk_mmio_unmap`, then `mk_device_release`
  (`src/handles/broker_handles_drop.rs:22`). The runtime reaches the IRQ grant through
  `handles.irq_grant_id()` for the interrupt poll (`src/handles/broker_handles_irq_grant_id.rs:20`) and
  the ABAR base through `handles.mmio_user_va()` (`src/handles/broker_handles_mmio_user_va.rs:20`).
- Each DMA region the port allocated is a `DmaRegion` that unmaps itself on drop with `mk_dma_unmap`
  (`src/engine/region.rs:46`), covered on the [engine](engine.md) page.

Because these owners drop when the process exits, and because the kernel revokes the claim and every grant
behind it on exit regardless, a crash or clean exit leaves no dangling device authority. The
[revocation](../../subsystems/hardware-broker/revocation.md) broker page describes the kernel side of that
teardown.

## Source map

```
  src/discover.rs                 find_ahci and is_candidate: the PCI storage/SATA/AHCI match
  src/setup/sequence.rs           setup::run: the one-shot bring-up and its failure unwinding
  src/setup/claim.rs              mk_device_claim wrapper, returns the claim epoch
  src/setup/pci.rs                mk_pci_config_write, sets the bus-master bit
  src/setup/mmio.rs               mk_mmio_map for BAR5, releases the device on failure
  src/setup/irq.rs                mk_irq_bind, unmaps ABAR and releases the device on failure
  src/setup/block_port.rs         picks the first present SATA port and calls init_port
  src/setup/driver.rs             the Driver struct: handles, regs, info, ports, block
  src/controller/enable.rs        enable_ahci: set GHC.AE, clear HBA_IS
  src/controller/info.rs          ControllerInfo::read and the port-count derivation
  src/controller/scan_ports.rs    the per-port snapshot and the present test
  src/controller/signature.rs     PxSIG classification into a port kind
  src/controller/port_info.rs     the PortInfo record shape
  src/handles/broker_handles.rs   BrokerHandles: the three grant ids and the device id
  src/handles/broker_handles_drop.rs   Drop: unbind irq, unmap mmio, release device
  src/regs/mmio.rs                the volatile 32-bit register wrapper
  src/constants/regs.rs           HBA and per-port register offsets, GHC.AE, signatures
  src/constants/pci.rs            AHCI_ABAR_BAR (5) and CLASS_BLOCK
  src/constants/port.rs           MAX_PORTS and the port-kind bytes
  docs/subsystems/hardware-broker/claim.md   the device-claim and epoch contract
  docs/subsystems/hardware-broker/mmio.md    the BAR-mapping contract
  docs/subsystems/hardware-broker/irq.md     the interrupt-bind contract
  docs/subsystems/hardware-broker/revocation.md   grant teardown on exit
```

Every reference above is verified against those trees.

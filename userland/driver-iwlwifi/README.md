# The Intel iwlwifi Driver Capsule

`capsule_driver_iwlwifi` is the NONOS Intel Wi-Fi hardware capsule: a signed ring-3 capsule that claims an
Intel wireless PCIe function, maps its registers, allocates a firmware staging buffer, and brings the device
through early power management. It does not run in the kernel and it does not touch the device through any
privileged kernel path. It reaches its controller only through the
[hardware broker](../../subsystems/hardware-broker/README.md), claiming the PCI function, mapping BAR0,
binding the device interrupt, and allocating one DMA staging region, all as brokered grants scoped to a
claim epoch. Everything above those grants, the register bring-up, the firmware TLV parse, and the small IPC
protocol, is ordinary userland code inside the capsule.

## Honest state first

This capsule is a real but partial Intel Wi-Fi driver, and it is worth being exact about the line it stops
at, because the [NIC driver model](../../subsystems/networking/drivers.md) sets an expectation this capsule
does not yet meet. What is real: it discovers a supported Intel function, takes exclusive ownership of it
through the broker, maps BAR0 and reads live registers, runs the APM (access-point-manager) clock bring-up
against the real `CSR_GP_CNTRL` register, reads the hardware revision, selects one of five bundled Intel
`.ucode` firmware blobs by device family, validates the Intel TLV firmware header, and copies the INIT,
runtime, and paging sections into a brokered DMA buffer with deterministic section records.

What is not real yet: the capsule never programs the flow-handler (FH) registers that would make the device
DMA the staged firmware out of that buffer, so the firmware is formatted into RAM but never handed to the
controller. There is no command queue, no TX or RX ring, no association, no scan, and no frame handoff to
`net.l2`. The alive-wait operation polls the interrupt-status register for the firmware alive bit, but since
nothing kicks the firmware transfer, that bit is not expected to be set on current hardware; the operation
is honest about that by returning a timeout errno when it is not seen. In the layered picture
`driver.iwlwifi0 -> wifi runtime -> net.l2 -> net.ip -> apps`, this capsule implements the first box up to
firmware staging and stops before the transfer that would light the firmware up. It is the mechanism a
Wi-Fi runtime is built on, brought to the point where the device is claimed, mapped, powered, and its
firmware selected and staged, not a working network interface.

The source under `userland/capsule_driver_iwlwifi/src/` is organized by concern, and this documentation
mirrors that structure one page per pillar so a page can be read beside the folder it describes.

## Identity

Everything the kernel and the service registry need to name and reach the driver comes from its `Capsule.mk`
and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-iwlwifi` | `Capsule.mk:5` |
| Service handle | `driver.iwlwifi0` | `Capsule.mk:6`, `src/hardware/iwlwifi_capsule/spawn.rs:31` |
| Namespace | `systems.nonos.driver.iwlwifi0` | `Capsule.mk:11` |
| Service endpoint | `service:4228:driver.iwlwifi0` | `Capsule.mk:12`, `spawn.rs:32` |
| Reply endpoint | `reply:4229:endpoint.4294967317` | `Capsule.mk:13`, `spawn.rs:33`, `spawn.rs:34` |
| Capability mask | `0xF8019` | `Capsule.mk:15` |
| Binary name | `driver_iwlwifi` | `Capsule.mk:9`, `Cargo.toml:18` |
| Kernel mirror | `src/hardware/iwlwifi_capsule` | `Capsule.mk:16`, `src/hardware/iwlwifi_capsule/spawn.rs` |

The reply endpoint number `4294967317` is `0x1_0000_0015`, the kernel reply inbox the spawn record names
(`REPLY_INBOX = "endpoint.4294967317"`, `src/hardware/iwlwifi_capsule/spawn.rs:33`). Note that the source
`README.md` in the capsule folder still calls out `MkMmioMap`, `MkIrqBind`, and `MkDmaMap` as if all three
are the only broker calls and describes an "APM clock-ready" bring-up; the shipped protocol and setup match
that, but the same source README's line about "the firmware path validates the Intel TLV header, ... stages
INIT, runtime, and paging sections into a brokered DMA window, and waits for the device alive interrupt
before higher Wi-Fi runtime work begins" reads as if the firmware is delivered to the device. It is not:
the staging writes into RAM only. This documentation follows the code.

The mask `0xF8019` decomposes bit by bit against `src/capabilities/types/bit.rs` (the enum is in `src/capabilities/types/defs.rs`):

```
  0x00008  IPC          bit()      8   types/bit.rs:26
  0x00010  Memory       bit()     16   types/bit.rs:27
  0x08000  DeviceEnum   bit()  32768   types/bit.rs:38
  0x10000  Driver       bit()  65536   types/bit.rs:39
  0x20000  Mmio         bit() 131072   types/bit.rs:40
  0x40000  Irq          bit() 262144   types/bit.rs:41
  0x80000  Dma          bit() 524288   types/bit.rs:42
  -------
  0xF8019  = 8 + 16 + 32768 + 65536 + 131072 + 262144 + 524288
```

The kernel spawn path requests exactly those seven capabilities and no others
(`src/hardware/iwlwifi_capsule/spawn.rs:50`), which matches the comment and value in the manifest
(`Capsule.mk:14`, `Capsule.mk:15`). This is the same seven-bit set the [NVMe driver](../driver-nvme/README.md)
holds: an ordinary app's `IPC` and `Memory`, plus the five hardware-broker authority bits the broker checks
before it issues any grant, `DeviceEnum` (enumerate devices), `Driver` (claim and release a device), `Mmio`
(map device registers), `Irq` (bind a device interrupt), and `Dma` (map DMA). It has no `Network` bit (4),
no `FileSystem` bit (64), and no graphics or raw-physmem authority; the firmware ships linked into the ELF,
so no filesystem authority is needed to load it. The security consequences of holding those bits are worked
through on the [bring-up](bring-up.md) page; the [claim](../../subsystems/hardware-broker/claim.md) page
documents how the broker enforces them.

## The three pillars

The capsule reads as three concerns, and the documentation is one page each. A client request enters through
the protocol and server (the operations page), which reports on a device that a one-time bring-up sequence
claimed, mapped, and powered (the bring-up page), whose firmware is selected, TLV-parsed, and staged into the
DMA buffer (the firmware page).

```
  client op   ->   server/protocol   ->   driver state + live registers
  NIWF IPC         parse, dispatch        Regs over BAR0, staged firmware

  one-time bring-up (setup):
  discover -> claim -> map BAR0 -> bind IRQ -> map DMA ->
  APM clock bring-up -> hw revision -> serve IPC

  firmware (on demand, into RAM only):
  select blob by family -> parse Intel TLV header -> stage INIT/RT/PAGING sections
```

| Page | Mirrors | What it covers |
|------|---------|----------------|
| [operations.md](operations.md) | `src/protocol/`, `src/server/` | The `NIWF` wire format, the request loop, the seven read-only ops, per-op payloads, the errno set, and the empty-body guard. |
| [bring-up.md](bring-up.md) | `src/setup/`, `src/discover.rs`, `src/init.rs`, `src/regs.rs`, `src/constants/`, `src/driver.rs` | Discovery, the four broker grants, the APM register bring-up, the hardware revision read, and the no-IOMMU caveat. |
| [firmware.md](firmware.md) | `src/firmware/` | The five bundled blobs, family selection, the Intel TLV header validation, the section staging into DMA, the alive poll, and where the FH transfer stops. |
| [contributing.md](contributing.md) | the whole tree | Where each concern lives, how to add an op, the build and sign steps, and the code standards. |
| [debugging.md](debugging.md) | runtime | The boot marker, the two setup exit codes, and the runtime errno set including the honest alive timeout. |

## Lifecycle

The capsule is `no_std`/`no_main`. `_start` initialises the heap, runs `setup::run`, and hands the built
`Driver` to the request server, which loops forever (`src/main.rs:35`). `setup::run` is the whole bring-up
(`src/setup/sequence.rs:19`); if the heap fails the process exits with code 1 and if setup fails it exits
with code 2, and it never serves a request (`src/main.rs:36`, `src/main.rs:40`). The kernel spawns it from
the bus-driver plan (`src/userspace/init/spawn_plan/drivers_bus.rs:24`) through
[verified spawn](../../security/capsules-and-trust.md), checking its signature and attestation and holding
its requested capabilities against its manifest ceiling before its ELF is mapped
(`src/hardware/iwlwifi_capsule/spawn.rs:59`). A successful spawn prints `[DRIVER-IWLWIFI] capsule spawned`
on the boot log; the [debugging](debugging.md) page covers what that and each exit code mean.

Once setup succeeds the capsule answers the seven read-only `NIWF` operations over IPC: a healthcheck,
device info, firmware info, RF state, DMA state, an on-demand firmware stage, and an alive wait. It does not
transmit or receive frames, associate, scan, or hand anything to the network stack; it reports and stages,
and that is the current boundary.

## Source map

```
  userland/capsule_driver_iwlwifi/src/main.rs        _start -> setup::run -> server::run; module list
  userland/capsule_driver_iwlwifi/src/protocol/      the NIWF wire format: header, ops, errno, limits, decode/encode
  userland/capsule_driver_iwlwifi/src/server/        the request loop and one handler per op
  userland/capsule_driver_iwlwifi/src/setup/         the bring-up sequence and the broker calls
  userland/capsule_driver_iwlwifi/src/discover.rs    the mk_device_list scan for the Intel Wi-Fi PCI function
  userland/capsule_driver_iwlwifi/src/init.rs        the APM clock bring-up and the InitState
  userland/capsule_driver_iwlwifi/src/regs.rs        Regs: volatile 32-bit access and poll over BAR0
  userland/capsule_driver_iwlwifi/src/driver.rs      the built Driver struct and the firmware stage method
  userland/capsule_driver_iwlwifi/src/firmware/      blob selection, TLV parse, section staging, alive poll
  userland/capsule_driver_iwlwifi/src/constants/     PCI ids, CSR offsets, GP_CNTRL bits, poll limits, FW API bounds
  userland/capsule_driver_iwlwifi/Capsule.mk         slug, handle, ports, capability mask, kernel mirror
  src/hardware/iwlwifi_capsule/                       the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/drivers_bus.rs        the DRIVER-IWLWIFI spawn entry
  src/capabilities/types/bit.rs                       the capability bit values behind the mask
```

Every reference above is verified against those trees.

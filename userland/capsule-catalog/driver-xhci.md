# capsule_driver_xhci (full reference)

`capsule_driver_xhci` is the USB 3 host-controller driver in the NONOS tree: a userspace capsule that
owns an xHCI controller end to end. It claims the PCI device through the hardware broker, maps the
controller's register window, binds its interrupt, allocates the controller's DMA structures, resets and
starts the controller, and then serves a request/reply IPC surface that USB class capsules (HID, mass
storage) call to enumerate and talk to devices. Nothing about USB lives in ring 0: the kernel grants
resources and revokes them, and every register write, ring, and descriptor read happens in this
CPL 3 capsule (`userland/capsule_driver_xhci/README.md:5`, `src/main.rs:36`).

It is a driver capsule, not an app-skeleton GUI app. Its entry point initialises the heap, runs the
one-shot bring-up (`setup::run`), and then hands the assembled `Driver` to a blocking server loop
(`src/main.rs:36`). The kernel spawns it on service port 4206 with a reply port on 4207, and its
capability mask is `0xF8019` (`Capsule.mk:13`, `Capsule.mk:16`). The source is
`userland/capsule_driver_xhci/`.

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

The capsule is the transport layer beneath every USB device on the machine. A USB HID capsule that wants
keyboard reports, or a mass-storage capsule that wants to read blocks, does not touch the controller: it
sends this capsule an IPC request, and this capsule drives the xHCI hardware to satisfy it. The split is
explicit in the README: this capsule owns controller mechanics only (bring-up, rings, event processing,
port status, the slot lifecycle), and USB class policy belongs to separate capsules above it
(`README.md:5`).

Structurally the capsule has two phases. The first is a one-shot setup that runs once at spawn and either
produces a fully initialised `Driver` or aborts the whole capsule with a negative errno
(`src/main.rs:40`, `src/setup/sequence.rs:34`). The second is a sequential server: one receive, one
dispatch, one reply, forever, with an interrupt service pass at the top of every iteration
(`src/server/runner.rs:33`). The `Driver` value threaded through the server holds everything the setup
built: the broker handles, the DCBAA, the scratchpad array, the DMA pool, the command and event rings,
the controller layout, and the per-slot table (`src/setup/driver.rs:22`).

The interrupt model is MSI-X. Setup binds one MSI-X vector through the broker
(`src/setup/irq_bind.rs:23`), and the server drains the event ring and acknowledges the interrupter on
each loop pass (`src/server/service_interrupts.rs:20`). The `Capsule.mk` header comment still describes
an INTx model with MSI-X deferred, which no longer matches the code (see
[code-vs-expectation](#a-note-on-target-names-and-comments) below).

## Identity

Everything the kernel and the service registry need to name and reach the driver comes from its
`Capsule.mk`.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-xhci` | `Capsule.mk:6` |
| Service handle | `driver.xhci0` | `Capsule.mk:7` |
| Domain | `systems.nonos` | `Capsule.mk:8` |
| Namespace | `systems.nonos.driver.xhci0` | `Capsule.mk:12` |
| Service endpoint | `service:4206:driver.xhci0` | `Capsule.mk:13` |
| Reply endpoint | `reply:4207:endpoint.4294967307` | `Capsule.mk:14` |
| Capability mask | `0xF8019` | `Capsule.mk:16` |
| Binary name | `driver_xhci` | `Capsule.mk:10` |
| Kernel feature | `nonos-capsule-driver-xhci` | `Capsule.mk:11` |
| Kernel mirror | `src/hardware/xhci_capsule` | `Capsule.mk:17` |

The reply endpoint id `4294967307` is `0x1_0000_000B`, the same constant the capsule hardcodes as its
fallback reply target for kernel-internal (pid 0) callers (`src/protocol/endpoint.rs:16`).

The mask `0xF8019` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x00001  CoreExec     bit()       1     types.rs:56
  0x00008  IPC          bit()       8     types.rs:59
  0x00010  Memory       bit()      16     types.rs:60
  0x08000  DeviceEnum   bit()   32768     types.rs:71
  0x10000  Driver       bit()   65536     types.rs:72
  0x20000  Mmio         bit()  131072     types.rs:73
  0x40000  Irq          bit()  262144     types.rs:74
  0x80000  Dma          bit()  524288     types.rs:75
  -------
  0xF8019  = 1 + 8 + 16 + 32768 + 65536 + 131072 + 262144 + 524288
```

Those eight bits are exactly the hardware-authority set a userspace driver needs and no more.
`DeviceEnum` lets it list PCI devices to find the controller; `Driver` lets it claim and release the
device; `Mmio` lets it map the register BAR; `Irq` lets it bind the interrupt; `Dma` lets it get
device-visible buffers; and `CoreExec`, `IPC`, and `Memory` are the base execute, message, and allocate
rights every capsule runs on (`src/capabilities/types.rs:35`). There is no `Pio` bit, which matches
xHCI being an MMIO-only controller, and no `FileSystem`, `Network`, `Graphics`, `InputSource`, `Admin`,
or `Debug` authority (`README.md:52`). The `Capsule.mk` comment spells the same mask out as
`IPC|Memory|Driver|DeviceEnum|Mmio|Irq|Dma` (`Capsule.mk:15`); note that it omits the `CoreExec` bit
that is also set.

## Operation reference

Requests carry the driver envelope: a 20-byte header of magic `NXHC` (`0x4E58_4843`), version `1`, op,
flags, a reserved word, a request id, and a payload length, followed by the payload
(`src/protocol/header.rs:16`, `src/protocol/encode.rs:17`, `src/protocol/decode.rs:17`). The server
validates the magic and version, rejects any frame whose declared payload length does not match the
bytes received or exceeds 10 bytes, and dispatches on the op (`src/server/runner.rs:41`,
`src/protocol/limits.rs:21`). Every reply begins with the same 20-byte header followed by a 4-byte
little-endian status word; a non-zero op body follows on success (`src/server/handlers/*`,
`src/protocol/limits.rs:16`). An unknown op, or an op whose body-length precondition fails, returns
`E_INVAL` (`src/server/dispatch.rs:40`).

The status word is one of the errno values in `src/protocol/errno.rs`: `0` success, `E_INVAL = -22`,
`E_IO = -5`, `E_NODEV = -19`, `E_AGAIN = -11` (`src/protocol/errno.rs:16`).

| Op | Opcode | Request body | Reply body (after status) | Errors |
|---|---|---|---|---|
| `OP_HEALTHCHECK` | `0x0001` | empty | none (status only) | `E_INVAL` if body non-empty |
| `OP_CONTROLLER_STATUS` | `0x0002` | empty | 56-byte controller snapshot | `E_INVAL` if body non-empty |
| `OP_PORT_STATUS` | `0x0003` | empty | 4-byte count header + 8 bytes/port | `E_INVAL` if body non-empty |
| `OP_ENABLE_SLOT` | `0x0004` | empty | 4 bytes, byte 0 = slot id | `E_IO`, `E_NODEV` |
| `OP_DISABLE_SLOT` | `0x0005` | 1 byte: slot id | none (status only) | `E_INVAL`, `E_IO` |
| `OP_ADDRESS_DEVICE` | `0x0006` | 2 bytes: slot id, root port | slot, port, speed, rsvd, EP0 MPS (LE), pad | `E_INVAL`, `E_NODEV`, `E_IO` |
| `OP_GET_DEVICE_DESCRIPTOR` | `0x0007` | 1 byte: slot id | 18-byte device descriptor | `E_INVAL`, `E_IO` |
| `OP_GET_CONFIG_DESCRIPTOR` | `0x0008` | 4 bytes: slot, index(=0), len (LE) | 2-byte count + raw config bytes | `E_INVAL`, `E_IO` |
| `OP_ALLOC_TRANSFER_RING` | `0x0009` | 6 bytes: slot, ep addr, max packet (LE), ep type | 4 bytes, byte 0 = DCI | `E_INVAL`, `E_IO` |
| `OP_CONTROL_TRANSFER` | `0x000B` | 10 bytes: slot, rsvd, bmRequestType, bRequest, wValue, wIndex, wLength | 2-byte count + returned data | `E_INVAL`, `E_IO` |
| `OP_INTERRUPT_IN` | `0x000E` | 4 bytes: slot, rsvd, length (LE) | 2-byte count + report bytes, or `E_AGAIN` | `E_INVAL`, `E_IO`, `E_AGAIN` |

Opcodes are defined in `src/protocol/ops.rs:16`. The gaps in the opcode space (`0x000A`, `0x000C`,
`0x000D`) are unassigned and fall through to the `E_INVAL` default arm.

Op detail, each cited:

- `OP_HEALTHCHECK` replies with a bare `status = 0`; it is the liveness probe
  (`src/server/handlers/health.rs:18`).
- `OP_CONTROLLER_STATUS` reads `USBSTS`, `USBCMD`, and `IMAN` live and returns a 56-byte snapshot: max
  slots and ports, max scratchpad and scratchpad page count, the three register words, the command-ring
  cycle bit, the total events drained, the DCBAA and scratchpad physical addresses, and the count of
  allocated slots (`src/server/handlers/controller_status.rs:23`, payload length 56 at
  `src/protocol/limits.rs:17`).
- `OP_PORT_STATUS` walks ports `1..=max_ports` (capped at `MAX_PORTS_REPORTED = 255`), reading each
  `PORTSC` and clearing its change bits, and returns a 4-byte count followed by an 8-byte record per
  port (port id + the raw `PORTSC` word) (`src/server/handlers/port_status.rs:22`,
  `src/protocol/limits.rs:18`).
- `OP_ENABLE_SLOT` issues an xHCI Enable Slot command, waits for the completion event, and on success
  marks the returned slot id allocated in the local table; if the table refuses it (out of range) it
  issues a compensating Disable Slot and returns `E_NODEV` (`src/server/handlers/enable_slot.rs:23`).
- `OP_DISABLE_SLOT` validates the slot is allocated, issues Disable Slot, and on success (if the slot
  was addressed) clears the DCBAA entry and drops the per-slot DMA resources, then marks the slot
  released (`src/server/handlers/disable_slot.rs:20`).
- `OP_ADDRESS_DEVICE` requires the slot allocated-but-not-yet-addressed and the port in range
  (`src/server/handlers/address_flow/slot_ready.rs:18`), resets the root port, then runs the full
  address flow and replies with slot, port, the xHCI speed id, and the EP0 max packet size
  (`src/server/handlers/address_device.rs:21`, `src/server/handlers/address_reply.rs:20`).
- `OP_GET_DEVICE_DESCRIPTOR` allocates an 18-byte DMA buffer, runs the EP0 `GET_DESCRIPTOR(Device)`
  control transfer against the slot's EP0 ring, and copies the raw descriptor into the reply
  (`src/server/handlers/device_descriptor.rs:23`).
- `OP_GET_CONFIG_DESCRIPTOR` accepts only index `0`, clamps the requested length to
  `CONFIG_DESCRIPTOR_MAX`, runs the EP0 configuration fetch, and prefixes the reply body with the
  actual byte count (`src/server/handlers/config_descriptor/handle.rs:23`,
  `src/server/handlers/config_descriptor/reply.rs:22`).
- `OP_CONTROL_TRANSFER` is the general EP0 control passthrough: it unpacks a USB setup packet
  (`bmRequestType`, `bRequest`, `wValue`, `wIndex`, `wLength`), allocates a `wLength` data buffer when
  non-zero, runs the three-stage control transfer, and returns the actual byte count plus any data read
  (`src/server/handlers/control_transfer/handle.rs:22`, `.../reply.rs:22`).
- `OP_ALLOC_TRANSFER_RING` converts an endpoint address to a device context index, configures the
  endpoint through a Configure Endpoint command, and returns the assigned DCI
  (`src/server/handlers/alloc_transfer_ring/handle.rs:23`, `src/slots/dci.rs:16`).
- `OP_INTERRUPT_IN` polls a slot's interrupt IN endpoint for one report up to `HID_REPORT_MAX = 8`
  bytes; a completed report returns the bytes, and a still-pending endpoint returns `E_AGAIN` in the
  status word so a HID capsule can poll without blocking the driver
  (`src/server/handlers/interrupt_in.rs:22`, `src/server/handlers/interrupt_in/reply.rs:23`,
  `src/protocol/limits.rs:36`).

## Architecture and bring-up

Bring-up is `setup::run`, a single ordered function that either returns a `Driver` or an `XhciError`
that `_start` turns into a negative-errno exit (`src/setup/sequence.rs:34`, `src/main.rs:40`). Each step
is a broker call or a controller register sequence, and the earlier hardware-authority steps roll their
predecessors back on failure.

The sequence, in order:

1. **Discover.** `find_xhci` calls `MkDeviceList` and scans for a PCI serial-bus / USB / xHCI device
   (class `0x0C`, subclass `0x03`, prog-if `0x30`) with a non-empty MMIO BAR0, returning its device id
   and BAR0 size (`src/discover.rs:27`). Absence is `DeviceNotFound`.
2. **Claim.** `MkDeviceClaim` takes exclusive ownership of the device and returns the claim epoch every
   later grant is checked against (`src/setup/claim.rs:18`).
3. **Bus master.** A PCI config write sets the Bus Master bit in the command register so the controller
   can DMA; failure releases the claim (`src/setup/pci.rs:21`, rollback at `src/setup/sequence.rs:37`).
   There is no explicit BIOS/OS handoff (xHCI USBLEGSUP) step in the code; the capsule does not walk the
   extended capabilities for a legacy-support semaphore.
4. **Map MMIO.** `MkMmioMap` maps BAR0 into the capsule, clamped to `min(bar0_size, 0x3000)` bytes so
   only the register window is exposed (`src/setup/mmio_map.rs:18`).
5. **Bind IRQ.** `MkIrqBind` binds one MSI-X vector; on failure it unmaps the MMIO grant and releases
   the claim before returning (`src/setup/irq_bind.rs:23`).
6. **Read layout.** `read_layout` first calls `refuse_unsupported`, which rejects a controller that is
   not 64-bit addressing capable (`AC64`) or reports zero slots (`src/controller/refuse_unsupported.rs:18`).
   It then reads `CAPLENGTH`, `RTSOFF`, and `DBOFF`, bounds each against the mapped length, and computes
   the operational, doorbell, and primary-interrupter base addresses plus max slots, max ports, max
   scratchpad, and the context size (`src/setup/layout.rs:25`).
7. **Halt.** Clear `USBCMD.RUN` and spin until `USBSTS.HCH` is set, else `HaltTimeout`
   (`src/controller/halt.rs:20`).
8. **Reset.** Set `USBCMD.HCRST` and spin until it self-clears, else `ResetTimeout`
   (`src/controller/reset.rs:20`); emits `[driver_xhci] reset ok`.
9. **Wait CNR.** Spin until `USBSTS.CNR` (Controller Not Ready) clears, else
   `ControllerNotReadyTimeout` (`src/controller/wait_cnr_clear.rs:20`); emits `[driver_xhci] cnr cleared`.
10. **Scratchpads.** Allocate the scratchpad buffer array sized to `max_scratchpad` from the DMA pool
    (`src/setup/sequence.rs:51`); emits `[driver_xhci] scratchpads ok`.
11. **DCBAA.** Allocate and zero the Device Context Base Address Array (`(max_slots + 1) * 8` bytes),
    write the scratchpad array pointer into slot 0, program `DCBAAP`, and set `CONFIG.MaxSlotsEn`
    (`src/controller/program_dcbaa.rs:20`); emits `[driver_xhci] dcbaa ok`.
12. **Command ring.** Allocate a 64-TRB command ring and program `CRCR` with its physical base and the
    ring cycle state (`src/controller/program_command_ring.rs:18`, `src/constants/ring.rs:17`); emits
    `[driver_xhci] cmd ring ok`.
13. **Event ring.** Allocate a 64-TRB event ring with a single-entry ERST, program the interrupter
    moderation to `4000` (`imod_program`), then program `ERSTSZ`, `ERDP`, `ERSTBA`, and set the
    interrupter's `IMAN.IE` (`src/setup/sequence.rs:59`, `src/controller/program_event_ring.rs:19`);
    emits `[driver_xhci] evt ring ok`.
14. **Start.** Clear `USBSTS.HSE`, then set `USBCMD.RUN` and `USBCMD.INTE`
    (`src/controller/start.rs:18`), and spin until `USBSTS.HCH` clears, else `StartTimeout`
    (`src/controller/wait_hc_running.rs:20`); emits `[driver_xhci] running`.
15. **No-op probe.** Enqueue a No-op command, ring doorbell 0, and wait for its completion event: a live
    proof that the command ring, doorbell, event ring, and interrupter all work end to end
    (`src/controller/issue_noop_and_wait.rs:22`); emits `[driver_xhci] noop ok` then
    `[driver_xhci] endpoint driver.xhci0 ready`.

`assemble` then packs the handles and structures into the `Driver` (`src/setup/sequence.rs:74`).

**The TRB model.** A Transfer Request Block is a 16-byte, 16-byte-aligned quad of `u32` fields
(`src/trb/base.rs:16`, `src/constants/ring.rs:16`). The TRB type constants are in
`src/constants/trb_kinds.rs`: Normal (1), Setup/Data/Status stages (2/3/4), Link (6), the command TRBs
Enable/Disable Slot (9/10), Address Device (11), Configure Endpoint (12), No-op (23), and the two event
types the driver consumes, Transfer Event (32) and Command Completion Event (33). Command and transfer
rings are built by the enqueue paths under `src/rings/`, and TRBs are read and written volatile through
`src/trb/read_volatile_at.rs` and `.../write_volatile_at.rs`.

**Command ring / event ring / DCBAA.** The command ring is where the driver posts controller commands;
`issue_enable_slot` enqueues an Enable Slot TRB, rings doorbell 0, and waits for the matching Command
Completion Event by physical pointer and success code (`src/controller/issue_enable_slot.rs:22`,
`src/controller/wait_command_completion.rs:25`). The event ring is where the controller posts
completions; the DCBAA is the array of per-slot device-context pointers the controller reads to find
each device's contexts. Input contexts are 33 entries and device contexts 32 entries, each entry being
the `context_size` (32 or 64 bytes) read from HCCPARAMS1.CSZ, so a 64-byte-context controller gets
64-byte-stride contexts (`src/contexts/size.rs:16`).

**Port enumeration.** `reset_port` first asserts Port Power (`PORTSC.PP`) if the firmware left it off,
which matters on real hardware where QEMU asserts connect immediately but a physical root port needs the
power-good settle (`src/controller/reset_port.rs:50`). If no device is connected (`PORTSC.CCS` clear) it
returns `NoDeviceOnPort`. Otherwise it clears change bits, drives Port Reset (`PORTSC.PR`), and spins
until reset-change and port-enabled are both set, else `PortResetTimeout`
(`src/controller/reset_port.rs:26`).

**IRQ handling.** The server drains and acks on every loop pass, not from an interrupt context. `service_interrupts`
calls `drain_events` (which copies up to `DRAIN_BATCH = 32` non-waiter TRBs off the event ring and
re-programs `ERDP`) and then `ack_irq`, which clears `IMAN.IP` if set and calls `MkIrqAck` on the IRQ
grant (`src/server/service_interrupts.rs:19`, `src/controller/drain_events.rs:30`,
`src/controller/ack_irq.rs:19`). Transfer and Command Completion events are deliberately left on the
ring for the synchronous waiters (`reserved_for_waiter`, `src/controller/drain_events.rs:44`), so a
command in flight is not consumed by the background drain.

## Protocol and IPC

The server loop is strictly sequential: one `MkIpcRecvFrom`, one dispatch, one reply, per iteration, with
the sender pid captured before dispatch so every handler reply routes back correctly
(`src/server/runner.rs:33`, `src/server/reply.rs:33`). A capsule caller (pid non-zero) gets a
correlation-matched `MkIpcReply` to its own reply inbox; a kernel-internal caller (pid 0) gets a
`MkIpcSend` to the fixed `KERNEL_REPLY_ENDPOINT` `0x1_0000_000B` (`src/server/reply.rs:37`). The eleven
client ops are the ones in the [operation reference](#operation-reference) above.

The broker syscalls the capsule calls, each cited, are the only way it reaches hardware:

```
  MkDeviceList     find_xhci enumerates PCI to locate the controller   src/discover.rs:29
  MkDeviceClaim    claim() takes the exclusive device claim + epoch     src/setup/claim.rs:19
  MkPciConfigWrite enable_bus_master sets PCI Bus Master                src/setup/pci.rs:22
  MkMmioMap        mmio_map maps BAR0 (clamped to 0x3000)               src/setup/mmio_map.rs:23
  MkIrqBind        irq_bind binds one MSI-X vector                      src/setup/irq_bind.rs:23
  MkDmaMap         DmaPool::alloc gets device-visible DMA buffers       src/dma/pool_alloc.rs:31
  MkIrqAck         ack_irq acknowledges the interrupter each loop       src/controller/ack_irq.rs:24
  MkMmioUnmap      irq_bind rollback unmaps MMIO on IRQ-bind failure    src/setup/irq_bind.rs:25
  MkDeviceRelease  rollback releases the claim on later setup failure   src/setup/sequence.rs:38
```

These map directly onto the four broker grant classes documented in the hardware broker subsystem: the
[device claim](../../subsystems/hardware-broker/claim.md) is the root authority, and the
[MMIO](../../subsystems/hardware-broker/mmio.md), [DMA](../../subsystems/hardware-broker/dma.md), and
[IRQ](../../subsystems/hardware-broker/irq.md) grants are all issued only to the claim holder and only
under the current claim epoch. Every one of the capsule's grant requests carries the `claim_epoch` it
received at claim time (`src/setup/sequence.rs:36`, threaded into `mmio_map`, `irq_bind`, and the DMA
pool), which is exactly the epoch value the broker re-checks at the head of each grant path.

## Security analysis

This capsule holds real hardware authority (`Driver`, `Mmio`, `Irq`, `Dma`, `DeviceEnum`), the most
powerful mask in the driver tree, so the interesting question is what bounds that authority. The answer
is that almost none of it is the capsule's own discretion: the broker grants it a slice of one device
and checks every grant against a claim.

**The claim is the root bound.** The capsule can only claim a device nothing else holds, and a claim is
exclusive: `MkDeviceClaim` refuses an already-claimed device, so no second capsule can be mapping this
controller's BARs or taking its interrupts underneath it
([claim.md](../../subsystems/hardware-broker/claim.md)). Every later grant it asks for carries the claim
epoch, and the broker rejects a stale epoch, so a grant handle cannot be replayed after the device
changes hands (`src/setup/claim.rs:18`, and the epoch check at the head of the MMIO, DMA, and IRQ paths).

**The MMIO grant is bounded to the BAR, minus the MSI-X table.** The capsule requests BAR0 clamped to
`0x3000` bytes (`src/setup/mmio_map.rs:19`), and the broker itself bounds the mapping to inside the BAR
and withholds the device's MSI-X table pages, so the capsule cannot program its own interrupt vectors by
writing the table directly ([mmio.md](../../subsystems/hardware-broker/mmio.md)). The kernel programs the
MSI-X entries during `MkIrqBind`; the capsule receives only a grant id and a vector
([irq.md](../../subsystems/hardware-broker/irq.md)). This is what makes the driver's interrupt
trustworthy: it drives the controller but cannot aim its interrupt at a victim.

**Bus-master DMA is real but broker-bounded and IOMMU-uncontained.** The capsule sets the PCI Bus Master
bit itself (`src/setup/pci.rs:22`), so once running the controller does bus-master DMA. The broker bounds
what the *capsule* may allocate: every DMA buffer comes from `MkDmaMap`, is zero-scrubbed before the
capsule sees it, and the USB-host class ceiling is 256 pages, so an over-request is refused with
`BadLengthForClass` rather than exhausting RAM ([dma.md](../../subsystems/hardware-broker/dma.md), and
the capsule's own per-grant page cap in `src/dma/pool_alloc.rs:27`). The honest gap, documented on the
DMA page, is that the IOMMU backend is not engaged in shipping builds, so the physical `device_addr` the
broker returns is a raw address and the controller could in principle DMA anywhere. DMA safety here rests
on the broker's software bounds plus the assumption of non-malicious controller hardware; enabling the
IOMMU backend is the path to closing that.

**The IPC surface is narrow and validated.** The server rejects a bad magic, a wrong version, a
payload-length mismatch, or an over-long payload before dispatch (`src/server/runner.rs:41`), and every
handler that takes a body checks its exact length before acting (for example the 1-byte disable, the
2-byte address, the 10-byte control transfer). An enable that the slot table cannot record is rolled
back with a compensating disable (`src/server/handlers/enable_slot.rs:36`), and the slot table is bounded
capsule-local state, not a controller structure a caller can overrun. USB descriptor parsing is
explicitly kept out of this capsule and the kernel: the driver returns raw descriptor bytes and lets the
class capsule parse them (`README.md:128`), so a malformed descriptor is a class-capsule problem, not a
driver or kernel memory-safety one.

**Isolation is the kernel's.** The capsule is a CPL 3 user binary verified and enrolled at spawn like
every other capsule; its mask is fixed in the signed manifest, and it speaks only the broker syscalls and
its own IPC service. A bug in ring handling or descriptor fetch cannot escalate past the one device it
claimed and the buffers the broker zeroed for it.

## How to contribute

The source lives at `userland/capsule_driver_xhci/`. The bring-up sequence is `src/setup/`, the
controller register and command primitives are `src/controller/` and `src/regs/`, the ring machinery is
`src/rings/`, the TRB builders are `src/trb/`, the IPC server and its per-op handlers are `src/server/`,
and the wire format is `src/protocol/`.

To add a new op:

1. Assign an opcode in `src/protocol/ops.rs:16` and, if the op takes a fixed body or returns a fixed
   payload, add its length constants to `src/protocol/limits.rs:16`.
2. Write the handler under `src/server/handlers/` as one module per op (a single-file handler like
   `src/server/handlers/port_status.rs`, or a folder with `handle.rs` / `reply.rs` / `transfer.rs` like
   `src/server/handlers/control_transfer/` when there is a data stage). A handler takes
   `(&mut Context, &Request, &[u8] body, &mut [u8] tx)`, validates the body length, does the work, and
   calls `reply_with_status` on error or `encode_response_header` + `write_status` + `reply::send` on
   success.
3. Wire the op into the match in `src/server/dispatch.rs:26`, gating any empty-body op with
   `if body.is_empty()` the way the existing status ops are.
4. If the reply can be larger than the current `TX_LEN`, widen the tx buffer sizing in
   `src/server/runner.rs:27`.

To build and sign the capsule, use the generated per-slug make targets (the slug is `driver-xhci`, so
the macro in `nonos-mk/capsule.mk:156` emits these; the file is included through
`userland/capsule_driver_xhci/Capsule.mk:19`):

```
  make nonos-mk-driver-xhci              build the capsule ELF
  make nonos-mk-driver-xhci-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-driver-xhci-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-driver-xhci-keys   check the per-capsule signing keys exist
```

The capsule's own README documents `make -B nonos-mk-driver-xhci` as the build command
(`README.md:165`). The `-sign` and `-verify` recipes are defined at `nonos-mk/capsule.mk:261` and `:263`.

For a running system that includes the driver, `make nonos-mk-driver-xhci-prod` builds a kernel with the
`microkernel-driver-xhci` feature and this capsule's signed artifacts embedded (`Makefile:975`). The USB
stack builds layer on it: `make nonos-mk-driver-usb-hid-prod` and `make nonos-mk-driver-usb-msc-prod`
both pull in `driver-xhci` artifacts as the transport (`Makefile:981`, `Makefile:986`). The boot harness
`make nonos-mk-boot-xhci` runs the round-trip test `tests/boot/xhci_round_trip.sh` (`Makefile:1399`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every fallible path returns an `XhciError` or an errno reply, never a panic,
and `_start` converts a setup error to a negative-errno exit rather than aborting,
`src/main.rs:40`); modular files, one unit per file, with `mod.rs` used only for re-exports (see
`src/controller/mod.rs`); and the AGPL header at the top of every source file, matching the header on
every existing module (`src/main.rs:1`). Architecture rule: xHCI must stay MMIO/IRQ/DMA broker-only and
must not use raw PIO or reach into kernel USB internals (`README.md:167`).

## Debugging

The first thing to confirm is that bring-up completed. Setup narrates its progress through
`nonos_libc::mk_debug` markers, printed in order, and the last one means the controller is live and the
service is registered (`src/setup/marker.rs:17`):

```
  [driver_xhci] reset ok                        HCRST self-cleared
  [driver_xhci] cnr cleared                     USBSTS.CNR dropped
  [driver_xhci] scratchpads ok                  scratchpad array allocated
  [driver_xhci] dcbaa ok                        DCBAA programmed, MaxSlotsEn set
  [driver_xhci] cmd ring ok                     CRCR programmed
  [driver_xhci] evt ring ok                     ERST/ERDP/ERSTBA programmed, IE set
  [driver_xhci] running                         USBCMD.RUN set, HCH cleared
  [driver_xhci] noop ok                         a No-op command completed end to end
  [driver_xhci] endpoint driver.xhci0 ready     server about to run
```

The missing marker names the step that blocked, and each blocking step has a distinct `XhciError`
(`src/error/xhci_error.rs`, surfaced as the negative exit in `src/main.rs:43`):

- **Reset never completes.** No `reset ok` marker: `HCRST` never self-cleared within the poll budget
  (`ResetTimeout`, `src/controller/reset.rs:28`). A halt that never sets `HCH` is the same failure one
  step earlier (`HaltTimeout`, `src/controller/halt.rs:31`).
- **Controller never runs.** `running` never prints: after setting `USBCMD.RUN` the controller never
  cleared `USBSTS.HCH` (`StartTimeout`, `src/controller/wait_hc_running.rs:27`). If it is stuck one
  step before, `USBSTS.CNR` never cleared (`ControllerNotReadyTimeout`,
  `src/controller/wait_cnr_clear.rs:27`).
- **No-op never completes.** `running` prints but `noop ok` does not: the command ring, doorbell, or
  interrupter is not wired (`CommandCompletionTimeout` or `CommandCompletionFailed`,
  `src/controller/wait_command_completion.rs:50`). This is the single best proof point, because it
  exercises the whole command/event path in one command.
- **Claim, MMIO, IRQ, or DMA grant refused.** No markers at all, and the capsule exits with a negative
  errno from `setup::run` (`src/main.rs:43`). This is a broker-side rejection; the broker's own staged
  `[MMIO]`, `[DMA]`, and IRQ-bind markers name which check failed (see the
  [claim](../../subsystems/hardware-broker/claim.md), [mmio](../../subsystems/hardware-broker/mmio.md),
  [dma](../../subsystems/hardware-broker/dma.md), and [irq](../../subsystems/hardware-broker/irq.md)
  pages), for example an `AlreadyClaimed` when a second xHCI capsule was spawned for the same device.
- **Server up but no ports show a device.** `OP_PORT_STATUS` returns entries but every `PORTSC` reads
  disconnected (`CCS` clear). On real hardware this is usually the port-power path: `reset_port` asserts
  `PORTSC.PP` when firmware left it off, and if the device still does not appear the settle budget was
  too short or the port is genuinely empty (`src/controller/reset_port.rs:50`). `OP_ADDRESS_DEVICE`
  against such a port returns `E_NODEV` (`src/server/handlers/address_device.rs:34`).
- **No devices enumerate past a slot.** `OP_ENABLE_SLOT` succeeds but `OP_ADDRESS_DEVICE` returns
  `E_IO`: the port reset or the Address Device command failed after the slot was enabled. Check the
  port state first (`OP_PORT_STATUS`) and confirm the slot was enabled but not yet addressed, which is
  the precondition `slot_ready` enforces (`src/server/handlers/address_flow/slot_ready.rs:18`).
- **An interrupt binds but never fires on real hardware.** This is the classic userspace-driver failure
  and it is not visible in this capsule's markers: the bind succeeded, `drain_events` finds nothing, and
  the event ring stays empty. The IRQ subsystem page documents the usual causes (an MSI-X redirect to a
  CPU that is not running, a masked line) and the diagnosis with `poll`
  ([irq.md](../../subsystems/hardware-broker/irq.md)). The QEMU proof path is
  `tests/boot/xhci_round_trip.sh` via `make nonos-mk-boot-xhci` (`Makefile:1399`).

### A note on target names and comments

Two in-tree comments no longer match the code, worth knowing when reading the source:

- The `Capsule.mk` header comment describes an INTx interrupt model with "MSI/MSI-X land later"
  (`Capsule.mk:1`), but `irq_bind` binds MSI-X today (`MK_IRQ_BIND_MSIX`, `src/setup/irq_bind.rs:23`).
- The kernel-mirror embed comment and the `.PHONY` list in the Makefile both reference a
  `nonos-mk-xhci` target (`src/hardware/xhci_capsule/embed.rs:18`, `Makefile:31`), but the target the
  build macro actually generates from `CAPSULE_SLUG := driver-xhci` is `nonos-mk-driver-xhci` (and its
  `-sign`/`-verify`/`check-...-keys` variants). The README's `make -B nonos-mk-driver-xhci` is the
  correct invocation (`README.md:165`). The `nonos-mk-xhci` name is a stale reference with no rule body.

## Source map

```
  src/main.rs                              _start -> heap, setup::run, server::run
  src/setup/sequence.rs                    the ordered bring-up (discover -> claim -> map -> irq -> reset -> start -> noop)
  src/setup/{claim,pci,mmio_map,irq_bind}.rs   the broker grant calls
  src/discover.rs                          find_xhci: PCI enumeration for the controller
  src/setup/driver.rs                      Driver: handles, DCBAA, scratchpads, rings, layout, slots
  src/controller/                          halt, reset, start, waits, DCBAA, rings, port reset, IRQ ack, event drain
  src/controller/layout.rs                 ControllerLayout (op/doorbell/interrupter bases, caps)
  src/regs/                                cap/op/runtime MMIO register accessors
  src/rings/{command,event,transfer}/      the ring state and enqueue paths
  src/trb/                                 TRB type, builders, command TRBs, volatile read/write
  src/contexts/                            input/output device contexts, 32/64-byte sizing
  src/slots/                               the per-slot table, DCI mapping, per-slot resources
  src/dma/                                 DmaPool over MkDmaMap and DmaRegion
  src/handles/                             BrokerHandles: device id, MMIO grant + VA, IRQ grant
  src/server/runner.rs                     the sequential recv/dispatch/reply loop
  src/server/dispatch.rs                   op -> handler routing
  src/server/handlers/                     one handler per op
  src/server/reply.rs                      pid-correlated MkIpcReply / fixed-endpoint MkIpcSend
  src/protocol/                            NXHC header, ops, errno, length limits, encode/decode
  Capsule.mk                               slug, handle, ports, capability mask, kernel mirror
  src/hardware/xhci_capsule                the kernel-side embed and verified spawn
  nonos-mk/capsule.mk                      the generated nonos-mk-driver-xhci[-sign|-verify] targets
  docs/subsystems/hardware-broker/         the claim/mmio/dma/irq grant paths this capsule calls
```

Every reference above is verified against those trees.

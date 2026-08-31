# The virtio-net Driver Capsule

`capsule_driver_virtio_net` is the virtio network backend: a signed userland capsule that claims one
virtio-net PCI device, drives its receive and transmit virtqueues over brokered DMA, and serves raw
Ethernet frames over IPC. It is the default NIC under QEMU, where `-device virtio-net-pci,netdev=net0`
backs the guest's network (`Makefile:286`, `Makefile:293`). It owns no ARP, no IP, no sockets, and no
routing; every layer above a raw frame lives in a capsule stacked on top of it. The frame consumer is
[net_core](../../subsystems/networking/stack.md), which calls this driver's `OP_TX_PACKET` and
`OP_RX_PACKET` directly over IPC (`userland/capsule_net_core/src/device/tx.rs:25`,
`userland/capsule_net_core/src/device/rx.rs:29`), and `net.l2` sits above net_core.

The source under `userland/capsule_driver_virtio_net/src/` splits into three concerns, and this
documentation is one page each so a page can be read beside the folder it describes: the client-facing
NNET protocol and its handlers, the device bring-up transaction with the broker grants it takes, and the
RX/TX virtqueue engine with the DMA rings and the virtio-net header.

## Identity

Everything the kernel and the broker need to name and reach the driver comes from its `Capsule.mk` and
its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-virtio-net` | `Capsule.mk:7` |
| Service handle | `driver.virtio_net0` | `Capsule.mk:8`, `src/hardware/virtio_net_capsule/spawn.rs:39` |
| Namespace | `systems.nonos.driver.virtio_net0` | `Capsule.mk:13` |
| Service endpoint | `service:4204:driver.virtio_net0` | `Capsule.mk:14`, `spawn.rs:40` |
| Reply endpoint | `reply:4205:endpoint.4294967305` | `Capsule.mk:15`, `spawn.rs:41` |
| Binary name | `driver_virtio_net` | `Capsule.mk:11` |
| Feature gate | `nonos-capsule-driver-virtio-net` | `Capsule.mk:12`, `spawn_plan/drivers_virtio_display.rs:35` |
| Capability mask | `0x1F8119` | `Capsule.mk:17` |
| Kernel mirror | `src/hardware/virtio_net_capsule` | `Capsule.mk:18` |

The reply endpoint id `4294967305` is `0x1_0000_0009`, slot 9 in the per-service reply numbering, which
the kernel-side client hard-codes as its `REPLY_INBOX` for the round trip it makes to the driver
(`src/hardware/virtio_net_capsule/client/transport.rs:28`). The kernel receives on it; the driver never
listens there. Note a stale-source discrepancy: that same kernel comment points at a userland file
`src/protocol/endpoint.rs`, but this capsule has no such file. The driver does not hold a fixed outbound
endpoint constant at all: its server receives with `mk_ipc_recv_from`, which yields the sender pid, and
replies straight back to that pid with `mk_ipc_reply` (`src/server/runner.rs:44`, `src/server/error.rs:24`).
The reply target is the caller, not a baked constant.

The mask `0x1F8119` decomposes bit by bit against `src/capabilities/types/bit.rs` (the enum is in
`src/capabilities/types/defs.rs`):

```
  0x000001  CoreExec     bit()       1   types/bit.rs:23
  0x000008  IPC          bit()       8   types/bit.rs:26
  0x000010  Memory       bit()      16   types/bit.rs:27
  0x000100  Debug        bit()     256   types/bit.rs:31
  0x008000  DeviceEnum   bit()   32768   types/bit.rs:38
  0x010000  Driver       bit()   65536   types/bit.rs:39
  0x020000  Mmio         bit()  131072   types/bit.rs:40
  0x040000  Irq          bit()  262144   types/bit.rs:41
  0x080000  Dma          bit()  524288   types/bit.rs:42
  0x100000  Pio          bit() 1048576   types/bit.rs:43
  ------
  0x1F8119  = 1 + 8 + 16 + 256 + 32768 + 65536 + 131072 + 262144 + 524288 + 1048576
```

The kernel spawn path requests `IPC | Memory | Debug | Driver | DeviceEnum | Mmio | Irq | Dma | Pio`,
with `Debug` coming through `serial_debug_cap()` (`src/hardware/virtio_net_capsule/spawn.rs:58`);
`CoreExec` is granted to every executable process. There is no `Network` bit (4) and no `FileSystem`
bit (64). The `Network` absence is deliberate and called out in the source: a
frame-level transport over IPC is not a network-service authority, and that authority belongs to a
future net-stack capsule layered on top (`Capsule.mk:1`, `spawn.rs:17`). The mask is the hardware-driver
envelope: the capsule can enumerate devices, claim one, map its registers by MMIO or PIO, bind its
interrupt, allocate DMA rings and buffers, and speak IPC. It cannot open a socket, spawn a process, or
reach a filesystem. `Mmio` and `Pio` are both present because the register BAR can be either
memory-mapped or a port range, and the driver takes whichever the device exposes; the [bring-up](bringup.md)
page covers that fork.

## The three pillars

The source is a discovery-and-setup front, a queue-and-DMA middle, and a protocol-and-server face. A
request arrives on the IPC endpoint, is parsed and dispatched by the server, either reads MMIO config or
moves a frame through a virtqueue ring, and the result is copied back into the reply.

```
  ipc in  ->  protocol/ + server/  ->  queue/ + rx.rs + tx.rs  ->  setup/ + init/ + regs/ (device)
  a message   parse NNET, dispatch    RX/TX rings, DMA, notify     claimed, negotiated
              on opcode                virtio-net header            device behind grants
```

| Page | Mirrors | What it covers |
|------|---------|----------------|
| [operations.md](operations.md) | `src/protocol/`, `src/server/` | The NNET wire format, the receive loop and opcode dispatch, the five operations and their bounds, and the reply-to-sender path. |
| [bringup.md](bringup.md) | `src/discover/`, `src/setup/`, `src/init/`, `src/regs/` | Device discovery, the ordered claim/register/irq/dma grant transaction and its rollback, the legacy virtio handshake and feature negotiation, and the MMIO-versus-PIO register abstraction. |
| [queues.md](queues.md) | `src/queue/`, `src/rx.rs`, `src/tx.rs` | The two split virtqueues, the RX prime and refill, the TX post/notify/wait path, the used-ring reader, and the 10-byte virtio-net header. |
| [contributing.md](contributing.md) | the whole tree | Where to work, how to add an operation, the build and sign steps, and the code standards. |
| [debugging.md](debugging.md) | runtime | The boot and setup-stage markers, the bring-up and per-request failure modes, and where to look for each. |

## Lifecycle

The capsule is a `no_std`/`no_main` binary. `_start` initialises the heap, then loops calling
`setup::run` until it returns a live `Driver`, yielding 64 times between attempts so a device that is
not yet claimable does not spin the CPU (`src/main.rs:41`). Once bring-up succeeds it checks that both
ring regions have a non-zero physical address, releasing and exiting if either is zero, then enters
`server::run`, which never returns (`src/main.rs:52`). There is no policy in the loop: the driver
discovers exactly one virtio-net device, negotiates it, allocates its DMA rings and buffers, reads the
MAC, primes RX, and then answers IPC.

Bring-up is an ordered transaction. Discover the device, claim it for the epoch every later grant must
quote, map its register BAR, bind its interrupt, allocate the four DMA regions (RX ring, RX buffers, TX
ring, TX buffer), negotiate features, read the MAC, program both queues, prime RX, drive `DRIVER_OK`,
kick RX, and acknowledge the interrupt (`src/setup/sequence.rs:27`). Each step depends on the grant the
previous one produced, and each DMA failure rolls back the grants already taken, so a failed bring-up
leaves the device unclaimed and no grants leaked. The [bring-up](bringup.md) page walks each step; the
broker side is documented in [claim](../../subsystems/hardware-broker/claim.md),
[mmio](../../subsystems/hardware-broker/mmio.md), [irq](../../subsystems/hardware-broker/irq.md), and
[dma](../../subsystems/hardware-broker/dma.md).

The driver is spawned through verified spawn: its signature and attestation are checked, its requested
capabilities are held against its manifest ceiling, and only then is its ELF mapped
(`src/hardware/virtio_net_capsule/spawn.rs:44`). The spawn is feature-gated behind
`nonos-capsule-driver-virtio-net` and runs from the virtio driver bring-up plan alongside the GPU
capsule (`src/userspace/init/spawn_plan/drivers_virtio_display.rs:35`). A successful spawn prints
`[DRIVER-VIRTIO-NET] capsule spawned` on the boot log; the [debugging](debugging.md) page covers what
each later marker means.

## Isolation

The virtio-net driver is a hardware-facing capsule, so its authority is real device authority, but it is
bounded on every axis the broker controls. It cannot open a socket, spawn a process, or reach a
filesystem; it can touch one claimed device and move frames over IPC. Every hardware action goes through
the broker, which validates it against the exclusive claim and its epoch: only one capsule can hold a
virtio-net device at a time, the MMIO grant can only map memory inside the device's own BAR, the IRQ
grant delivers on a kernel-owned vector, and the DMA regions are allocated and zero-scrubbed by the
broker before the capsule sees them ([claim](../../subsystems/hardware-broker/claim.md),
[mmio](../../subsystems/hardware-broker/mmio.md), [irq](../../subsystems/hardware-broker/irq.md),
[dma](../../subsystems/hardware-broker/dma.md)).

The request bounds are the driver's own line of defence against a client. A transmit requires
`payload_len` to equal the body length, a non-empty body no larger than `MAX_ETHERNET_FRAME` = 1514, and
a body within `MAX_TX_PAYLOAD_BYTES`, else `E_MSGSIZE` or `E_INVAL` (`src/server/handlers/tx_packet.rs:24`).
On receive, the frame slice handed back is clamped to the buffer capacity minus the virtio-net header
before it is formed, so a device that reports an oversized length cannot make the driver read past the
RX buffer (`src/rx.rs:52`). A message that fails to decode is answered `E_INVAL` and never reaches a
handler (`src/server/runner.rs:51`). The [operations](operations.md) page details each bound.

The honest caveat is the IOMMU. NONOS carries an `IommuDomain` abstraction but its hardware backend is
behind the `nonos-arch-iommu` feature and is not engaged in the shipping builds, so the device physical
address the broker hands back is a raw physical address and a malicious or buggy device could in
principle DMA to any physical address regardless of the grant
([dma](../../subsystems/hardware-broker/dma.md)). The broker bounds what the capsule may allocate and
program; it does not yet bound what the device does once running. On the kernel side, the in-kernel
client that talks to this endpoint is itself gated: only a caller holding `CAP_DRIVER` may reach the
frame surface (`src/hardware/virtio_net_capsule/capability.rs:30`).

## Source map

```
  src/main.rs                         _start: heap init, retry setup::run, ring-phys check, server::run
  src/discover/                       find_virtio_net: match vendor/device, pick register BAR
  src/setup/                          the bring-up transaction (claim, regs, irq, four DMA maps) and rollback
  src/init/                           negotiate, program_queue, driver_ok: the legacy virtio handshake
  src/regs/                           MMIO vs PIO register access behind the Regs abstraction
  src/queue/                          the RX/TX queue state, prime, refill, post, and used ring
  src/rx.rs                           take_one: pull a frame past the virtio-net header
  src/tx.rs                           send: copy, post, notify, wait, ack
  src/protocol/                       the NNET header, ops, errno, and limits
  src/server/                         the receive loop and the health/link/mac/tx/rx handlers
  src/constants/                      frame, pci, queue, regs, and status constants
  Capsule.mk                          slug, handle, ports, capability mask, kernel mirror
  src/hardware/virtio_net_capsule/    the kernel-side embed, verified spawn, gate, and mirrored protocol
  src/userspace/init/spawn_plan/drivers_virtio_display.rs   the driver spawn entry and boot marker
  src/capabilities/types/bit.rs       the capability bit values the mask decomposes against
```

Everything here is drawn from `userland/capsule_driver_virtio_net/` (the capsule source and its
`Capsule.mk`), `userland/capsule_net_core/` (the frame consumer), `src/capabilities/types/bit.rs`, and the
kernel mirror under `src/hardware/virtio_net_capsule/`. Every reference above is verified against those
trees.

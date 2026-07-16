# Debugging capsule_driver_virtio_net

This page lists the log markers the driver and its boot path emit, and the concrete failure modes with
where to look for each. For the client protocol see the [operations](operations.md) page, for the device
side the [bring-up](bringup.md) page, and for the frame engine the [queues](queues.md) page.

## Log markers

The first thing to confirm is that the capsule ran. On a successful boot the kernel logs
`[DRIVER-VIRTIO-NET] capsule spawned`: the boot plan names the driver, calls
`spawn_driver_virtio_net_capsule`, and the `Ok` arm logs `boot_log::ok(prefix, "capsule spawned")`
(`src/userspace/init/spawn_plan/drivers_virtio_display.rs:38`, `src/userspace/init/capsule_boot/run.rs:29`).
If that line is absent the capsule never started, and the `Err` arm logged the failure through
`boot_log::error` instead (`src/userspace/init/capsule_boot/run.rs:32`), which is the usual signature,
manifest, or capability failure. Note the spawn lives in the virtio display plan alongside the GPU
capsule, and both are feature-gated: without `nonos-capsule-driver-virtio-net` the spawn is a no-op
(`src/userspace/init/spawn_plan/drivers_virtio_display.rs:45`).

The load-error debug tag baked into the spawn spec is `[DRIVER-VIRTIO-NET] load_elf_executable error:`,
so an ELF or mapping failure surfaces under that prefix (`src/hardware/virtio_net_capsule/spawn.rs:66`).

Once the capsule is running, `setup::run` emits a `[net-setup]` stage marker over `mk_debug` before each
step: `find`, `claim`, `regmap`, `irq`, `dma`, `negotiate`, `queues`, `irq-ack`, and `done`
(`src/setup/sequence.rs:28`, `src/setup/stage.rs:19`). The last marker printed names the step a stuck or
failed bring-up died on.

## Failure modes

### Bring-up never completes

`_start` retries `setup::run` forever, yielding 64 times between attempts, so a driver that spawned but
serves nothing is stuck in bring-up (`src/main.rs:41`). The `[net-setup]` markers tell you where. The
usual causes are broker refusals: the claim is already taken by another instance, or the device is not in
the broker table at all. The broker narrates each on the console, and a `NONOS_DEVICE_CENSUS=1` build
renders the device table so you can confirm the NIC is present before any driver runs
([claim](../../subsystems/hardware-broker/claim.md)). A DMA refusal prints a `[DMA]` line naming the
failed check ([dma](../../subsystems/hardware-broker/dma.md)). If discovery itself finds nothing,
`setup::run` returns `no virtio-net device` (`src/setup/sequence.rs:29`). Unlike some drivers, discovery
does not skip a device with no interrupt pin or an unrouted line, precisely because q35 firmware leaves
`irq_line` at `0xFF` and the driver prefers MSI-X; a present NIC is matched even when its legacy line is
unrouted (`src/discover/find_virtio_net.rs:32`).

### Bring-up aborts on negotiation or queue setup

`negotiate` returns `virtio-net: features-ok rejected` if the device cleared `FEATURES_OK` after the
driver set it, usually a feature-set mismatch; on that path the device's `FAILED` bit is already set
(`src/init/negotiate.rs:37`). `program_queue` returns `virtio-net: invalid queue`, also after setting
`FAILED`, if a queue reported a zero max size, if the ring physical address was not page-aligned, or if
its page frame did not fit 32 bits (`src/init/program_queue.rs:29`). An IRQ bind that could not fall back
to MSI-X returns `irq bind failed` or, if its own rollback failed, `irq bind rollback failed`
(`src/setup/irq.rs:38`). A DMA map failure returns one of the `dma map failed (...)` strings naming the
region, or a `dma rollback failed (...)` string if the unwind itself failed
(`src/setup/dma/rx_queue.rs:35`). The final `mk_irq_ack` failure returns `virtio-net: irq ack failed`
(`src/setup/sequence.rs:53`).

### The driver exits instead of serving

After bring-up, `_start` checks that both ring regions have a non-zero physical address; if either is
zero it releases every grant and exits with code 3, or code 4 if the release itself failed
(`src/main.rs:52`). A driver that spawned, completed bring-up, and then vanished with one of those codes
mapped a ring region to a zero physical address, which points at the DMA grant rather than the driver
logic.

### A request returns an error status

The status word in the reply is the diagnosis (`src/protocol/errno.rs:24`).

- `E_MSGSIZE` (-90): on transmit, `payload_len` did not match the body length, or the frame exceeded the
  transmit payload cap (`src/server/handlers/tx_packet.rs:24`, `src/server/handlers/tx_packet.rs:30`).
- `E_INVAL` (-22): a bad opcode, a header that failed to decode, or a transmit body that was empty or
  larger than `MAX_ETHERNET_FRAME` = 1514 (`src/server/runner.rs:63`, `src/server/handlers/tx_packet.rs:27`).
- `E_AGAIN` (-11): `OP_RX_PACKET` found the RX ring empty. This is an ordinary state, not a fault: the
  caller polls again (`src/server/handlers/rx_packet.rs:34`).
- `E_IO` (-5): the transmit path returned an error, either a completion timeout or a failed interrupt ack
  (`src/server/handlers/tx_packet.rs:35`, `src/tx.rs:65`).

### A transmit hangs

`tx::send` gives up with `Timeout` after `MAX_YIELDS` = 200000 yields, either waiting for a free ring
slot under backpressure or waiting for the used index to reach the completion target (`src/tx.rs:39`,
`src/tx.rs:61`). A `Timeout` surfaces to the handler as an `E_IO` reply, but a persistent hang before any
reply points at the IRQ binding or the device not completing rather than at request parsing. The transmit
wait is a used-index poll, so a completion is caught even if the interrupt is coalesced; a hang means the
used index never advanced.

### Received frames never arrive

`OP_RX_PACKET` returns `E_AGAIN` whenever the used index equals the driver's `last_used`
(`src/rx.rs:41`). If frames are on the wire but every poll returns `E_AGAIN`, confirm RX was primed at
bring-up (`RxQueue::prime` publishes all buffers) and that the RX notify is firing: the handler re-kicks
`LEG_QUEUE_NOTIFY` after every poll (`src/queue/post.rs:24`, `src/server/handlers/rx_packet.rs:28`). A
frame whose device-reported length is at most the 10-byte virtio-net header yields an empty payload rather
than a fault, so an all-empty stream can also mean the device is delivering header-only descriptors
(`src/rx.rs:52`).

### A reply is truncated or wrong-sized

The payload-carrying replies send an exact length, not the whole buffer: link sends
`RESP_HDR_LEN + STATUS_LEN + 1`, mac sends `RESP_HDR_LEN + STATUS_LEN + 6`, and rx sends
`RESP_HDR_LEN + STATUS_LEN + 4 + frame_len` (`src/server/handlers/link_status.rs:35`,
`src/server/handlers/mac_address.rs:29`, `src/server/handlers/rx_packet.rs:47`). If an rx reply looks
short, the 4-byte length prefix is authoritative; the frame copy is bounded to the RX buffer through
`take_one`, so a mismatch is a length-prefix reading issue, not an overrun (`src/rx.rs:52`).

## Source map

```
  src/userspace/init/spawn_plan/drivers_virtio_display.rs   the driver spawn entry and boot marker
  src/userspace/init/capsule_boot/run.rs               [DRIVER-VIRTIO-NET] capsule spawned / error path
  src/hardware/virtio_net_capsule/spawn.rs             the load-error debug tag
  src/main.rs                         the forever-retry loop and the ring-phys exit codes
  src/setup/sequence.rs               the [net-setup] markers and no virtio-net device / irq ack failed
  src/setup/stage.rs                  the stage marker helper
  src/setup/irq.rs                    the irq bind failure strings
  src/setup/dma/rx_queue.rs           the dma map/rollback failure strings
  src/init/negotiate.rs               features-ok rejected
  src/init/program_queue.rs           invalid queue
  src/discover/find_virtio_net.rs     the no-irq-filter discovery comment
  src/protocol/errno.rs               the reply error codes
  src/server/runner.rs                the unknown-opcode E_INVAL
  src/server/handlers/tx_packet.rs    the transmit bounds checks
  src/server/handlers/rx_packet.rs    the E_AGAIN empty-ring path
  src/rx.rs                           the bounded frame slice and the empty/header-only case
  src/tx.rs                           the Timeout and the ack failure
```

Every reference above is verified against those trees.

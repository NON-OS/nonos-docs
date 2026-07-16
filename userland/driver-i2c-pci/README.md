# capsule_driver_i2c_pci (full reference)

`capsule_driver_i2c_pci` is the Intel LPSS I2C host controller driver in the NONOS tree: the PCI bus
master that i2c-HID devices ride on. It is a userspace driver capsule. It owns exactly one Intel LPSS
DesignWare I2C function, which it claims through the hardware broker, maps BAR0 for, binds the interrupt
line for, and drives as an I2C master. Everything above the bus, HID report parsing, touchpad gestures,
sensor policy, input focus, stays in higher capsules; this driver only moves bytes on the wire and
reports controller state (`userland/capsule_driver_i2c_pci/Capsule.mk:1`, `README.md:16`).

The capsule is honest about its scope. The I2C stack is a partial slice: discovery is Intel and PCI
only, ACPI-enumerated controllers are not matched, and transfers are polled rather than interrupt
driven. Those gaps are documented in [Architecture and bring-up](#architecture-and-bring-up) and
[Security analysis](#security-analysis), not hidden.

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

The capsule is `no_std`/`no_main`. `_start` initialises the heap, runs the one-shot bring-up
`setup::run`, and on success hands the resulting `Driver` to the blocking IPC server `server::run`; a
bring-up failure exits with code 1 (`src/main.rs:19`). There is no runtime path that reaches hardware
except through the broker grants obtained during bring-up.

The role is deliberately narrow. This capsule is the controller driver, not the device driver: it
enumerates PCI, finds a supported Intel LPSS I2C function, claims it, maps its register window, binds its
interrupt, brings the DesignWare master out of reset with a valid clock program, and then answers IPC
requests that read controller state or run a bounded master transfer on the bus. The i2c-HID capsule is
its client and holds the descriptors and reports; this driver holds only the controller
(`userland/capsule_driver_i2c_pci/README.md:5`, `README.md:91`). The `Driver` value that the server
carries is the entire runtime state: the claimed device id, the PCI device id, the claim epoch, the MMIO
and IRQ grant ids, the bound vector, the source clock, the family name, the DesignWare component
type/param read at init, the cached enable/status words, and the register accessor
(`src/driver.rs:3`).

## Identity

Everything the kernel and the service registry need to name and reach the driver comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-i2c-pci` | `Capsule.mk:6` |
| Service handle | `driver.i2c_pci0` | `Capsule.mk:7`, `src/hardware/i2c_pci_capsule/spawn.rs:23` |
| Namespace | `systems.nonos.driver.i2c_pci0` | `Capsule.mk:12` |
| Service endpoint | `service:4230:driver.i2c_pci0` | `Capsule.mk:13`, `spawn.rs:24` |
| Reply endpoint | `reply:4231:endpoint.4294967318` | `Capsule.mk:14`, `spawn.rs:25`, `spawn.rs:26` |
| Capability mask | `0x78019` | `Capsule.mk:16` |
| Binary name | `driver_i2c_pci` | `Capsule.mk:10` |
| Kernel mirror | `src/hardware/i2c_pci_capsule` | `Capsule.mk:17` |

The mask `0x78019` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x00008  IPC          bit()      8   types.rs:59
  0x00010  Memory       bit()     16   types.rs:60
  0x08000  DeviceEnum   bit()  32768   types.rs:71
  0x10000  Driver       bit()  65536   types.rs:72
  0x20000  Mmio         bit() 131072   types.rs:73
  0x40000  Irq          bit() 262144   types.rs:74
  -------
  0x78019  = 8 + 16 + 32768 + 65536 + 131072 + 262144
```

The kernel spawn path requests exactly those six capabilities and no others
(`src/hardware/i2c_pci_capsule/spawn.rs:42`). `DeviceEnum` is enumerate-only, `Driver` lets the capsule
claim and release a device, `Mmio` lets the claim holder map a slice of a BAR, and `Irq` lets it bind
the device interrupt (`src/capabilities/types.rs:34`). There is no `CoreExec` in the mask (this is not
a top-level app), no `DMA`, no `PIO`, no `FileSystem`, no `Network`, and no display or input-focus
capability. That is the whole basis of the security analysis below: the driver can enumerate devices,
claim one, map its registers, bind its IRQ, and speak IPC, and nothing else.

The kernel mirror embeds the signed ELF, id cert, manifest, and attestation trailer and spawns the
capsule at boot through the bus-driver spawn plan (`src/hardware/i2c_pci_capsule/embed.rs:9`,
`src/userspace/init/spawn_plan/drivers_bus.rs:38`).

## Operation reference

The server is a blocking receive loop. It allocates a fixed receive and transmit buffer of `HDR_LEN +
IPC_PAYLOAD_MAX` bytes (20 + 256), receives from the service inbox with the sender pid, parses the
header, and dispatches on the opcode (`src/server/runner.rs:14`). A message that fails to parse or comes
from pid 0 is dropped silently (`runner.rs:20`). Every reply is the 20-byte header plus a 4-byte signed
status word plus an optional body (`src/protocol/encode.rs:3`).

The wire header is the shared `NI2C` capsule envelope: magic `0x4E49_3243` ("NI2C"), version `1`, a
2-byte opcode, an 8-byte request id, and a 4-byte body length, 20 bytes total
(`src/protocol/header.rs:1`). The decoder rejects a short buffer, a wrong magic or version, or a body
length that runs past the buffer (`src/protocol/decode.rs:3`). All multi-byte integers are little-endian.

Six opcodes are defined (`src/protocol/ops.rs:1`). The four fixed-width read operations require an empty
body; a non-empty body on one of them falls through to `E_INVAL`, and an unknown opcode with an empty
body replies `E_BAD_OP` (`src/server/runner.rs:36`, `runner.rs:46`).

Status words (`src/protocol/errno.rs:1`):

```
  E_OK        0     success
  E_BUSY    -16     controller master still active before the transfer
  E_INVAL   -22     malformed request body or out-of-range address/length
  E_BAD_OP  -38     unknown opcode
  E_TIMEOUT -110    controller or transfer did not complete in the iteration budget
  E_NACK   -121     device did not acknowledge (DesignWare TX abort)
```

### OP_HEALTHCHECK (opcode 1)

Request: empty body. Reply: `E_OK` with a single byte `1` (`src/server/handlers/health.rs:4`). This is
the liveness probe; it reads no registers.

### OP_CONTROLLER_INFO (opcode 2)

Request: empty body. Reply: `E_OK` with a 64-byte body carrying the cached identity, with no register
reads (`src/server/handlers/controller.rs:5`):

```
  [ 0.. 8)  device_id     u64   broker device id
  [ 8..10)  pci_device    u16   PCI device id
  [10..14)  clock_hz      u32   source clock
  [14..22)  claim_epoch   u64   claim epoch
  [22..30)  mmio_grant    u64   MMIO grant id
  [30..38)  irq_grant     u64   IRQ grant id
  [38..42)  irq_vector    u32   bound broker vector
  [42..64)  family        utf8  family name, up to 22 bytes
```

### OP_REGISTER_SNAPSHOT (opcode 3)

Request: empty body. Reply: `E_OK` with a 40-byte body of ten little-endian `u32` values
(`src/server/handlers/snapshot.rs:6`). The first four are the values cached at init (`comp_type`,
`comp_param`, `enabled`, `status`); the remaining six are live reads of `IC_CON` (0x00), `IC_INTR_MASK`
(0x30), `IC_RAW_INTR_STAT` (0x34), `IC_TXFLR` (0x74), `IC_RXFLR` (0x78), and `IC_ENABLE` (0x6C)
(`src/constants/mod.rs:6`). These are side-effect-free status reads.

### OP_TIMING_INFO (opcode 4)

Request: empty body. Reply: `E_OK` with a 28-byte body of seven `u32` values: the source clock, then
live reads of the standard-mode SCL high/low counts (`IC_SS_SCL_HCNT` 0x14, `IC_SS_SCL_LCNT` 0x18), the
fast-mode counts (`IC_FS_SCL_HCNT` 0x1C, `IC_FS_SCL_LCNT` 0x20), and the RX/TX FIFO thresholds
(`IC_RX_TL` 0x38, `IC_TX_TL` 0x3C) (`src/server/handlers/timing.rs:6`). This is how a client reads back
the clock program the driver applied during bring-up.

### OP_TRANSFER (opcode 5)

Request: an 8-byte fixed head followed by the write bytes (`src/server/handlers/transfer.rs:22`):

```
  [0]      addr        u8    7-bit target address
  [1]      reserved    u8
  [2..4)   write_len   u16
  [4..6)   read_len    u16
  [6..8)   flags       u16   bit 0 = FLAG_RESTART_ON_READ
  [8..]    write       write_len bytes
```

The parser rejects a body shorter than 8 or one whose length is not exactly `8 + write_len` with
`E_INVAL` (`transfer.rs:23`, `transfer.rs:30`). `FLAG_RESTART_ON_READ` (bit 0) inserts a repeated-start
before the first read command of a write-then-read transaction
(`src/transaction/types/flags.rs:16`, `src/transaction/engine/read_cmd.rs:21`). Write and read are each
bounded to 64 bytes; a request over that bound is `E_INVAL`
(`src/protocol/limits.rs:2`, `src/transaction/types/valid_lengths.rs:18`).

Reply on success: `E_OK` with a body whose first `u16` is the read length, then a 4-byte DesignWare
abort source (zero on a clean transfer), then the read bytes at offset 8
(`src/server/handlers/transfer.rs:36`). Errors map straight from the transaction engine: `E_BUSY`,
`E_TIMEOUT`, `E_NACK`, `E_INVAL` (`transfer.rs:44`).

### OP_PROBE (opcode 6)

Request: a single byte, the 7-bit address; a body that is not one byte or an address over `0x7F` is
`E_INVAL` (`src/server/handlers/probe.rs:7`). The probe runs a one-byte read transfer and maps the
outcome: a completed transfer replies `E_OK` with `[1]` (present), a NACK replies `E_OK` with `[0]`
(absent), and any other error is surfaced as its errno
(`src/transaction/engine/probe.rs:21`, `src/server/handlers/probe.rs:11`). The distinction matters: a
NACK is a definite "nobody home", not a bus failure.

## Architecture and bring-up

Bring-up is one linear sequence (`src/setup/sequence.rs:9`): discover, claim, map, bind, initialise,
ack, then build the `Driver`. Each step that can fail unwinds the grants it already took, so a partial
bring-up leaves no dangling claim or mapping.

### Discovery (Intel and PCI only)

`find_controller` calls the kernel `MkDeviceList` syscall into a 128-entry buffer and scans it
(`src/discover.rs:34`). A record qualifies only if it is an Intel vendor (`0x8086`) PCI device of class
serial bus (`0x0c`) whose PCI device id is in the driver's static table, and it must have a real
interrupt pin and line and a non-zero MMIO BAR0 (`src/discover.rs:41`, `discover.rs:60`). The device
table maps PCI ids to a family name and a source clock, spanning Skylake-era Sunrise Point through
Meteor Lake, plus Broxton, Gemini Lake, and Jasper Lake, with clocks of 120 MHz on the older parts and
100 MHz on Tiger Lake and later (`src/constants/mod.rs:28`).

This is the first honest gap. Discovery is PCI and Intel only. A controller that is present on the
platform but enumerated through ACPI rather than PCI, or a PCI id not in the table, is not matched and
the capsule exits with `controller not found`. The README states ACPI-enumerated device matching as an
explicit non-goal of this slice and a named next step (`README.md:113`, `README.md:132`). On a laptop
where the LPSS controllers are described in ACPI and never surface a matching PCI function, this driver
will not find them.

### Claim, map, bind

The three broker steps run in order against the discovered device id and the claim epoch:

- **Claim.** `mk_device_claim` returns the epoch; a non-positive result is `device claim failed`
  (`src/setup/claim.rs:3`). The epoch is the anti-stale token every later grant carries; see the
  [device claim](../../subsystems/hardware-broker/claim.md) page.
- **Map.** `mk_mmio_map` maps BAR0 at offset 0, rounding the BAR0 size up to a page boundary, and on
  failure releases the claim before returning `mmio map failed` (`src/setup/mmio.rs:6`). The broker
  clamps the mapping to the BAR and withholds any MSI-X table pages; see the
  [MMIO](../../subsystems/hardware-broker/mmio.md) page. The returned user VA becomes the register base.
- **Bind.** `mk_irq_bind` binds the device's PCI interrupt line, and on failure unmaps the MMIO grant
  and releases the claim before returning `irq bind failed` (`src/setup/irq.rs:5`). This is a legacy
  INTx bind: the bind request passes the `irq_line` with zero flags, so the broker takes the INTx path,
  not MSI-X (`setup/irq.rs:7`; the two bind modes are on the
  [IRQ](../../subsystems/hardware-broker/irq.md) page).

### Controller bring-up and the clock program

`bring_up` (`src/init/mod.rs:14`) resets the DesignWare master into a known state. It disables the
controller and spins on `IC_ENABLE_STATUS` until the enable bit clears, failing with `controller
disable timeout` if it never does (`src/init/mod.rs:47`). While disabled it writes `IC_CON` to select
master mode, fast speed, restart-enable, and slave-disable (`init/mod.rs:16`,
`src/constants/mod.rs:65`), programs the SCL clock, zeroes the RX and TX FIFO thresholds, masks all
interrupts (`IC_INTR_MASK` = 0), and reads `IC_CLR_INTR` once to clear pending state
(`init/mod.rs:24`). Finally it reads and caches the component type, component param, enable status, and
status registers, which are the values `OP_CONTROLLER_INFO` and the first four words of the snapshot
report (`init/mod.rs:28`).

The clock program is the load-bearing detail. The DesignWare master emits no SCL clock at all until the
HCNT/LCNT count pairs are programmed, and those registers are writable only while the controller is
disabled (`init/mod.rs:36`). `program_clock` writes the standard and fast SCL high/low pairs and the
SDA hold time, all derived from the discovered source clock (`init/mod.rs:37`). The counts follow the
Linux dw_i2c formulas, `HCNT = clk_hz * tHIGH_ns / 1e9 - 3` and `LCNT = clk_hz * tLOW_ns / 1e9 - 1`,
with the bus timing budgets from the I2C specification; the arithmetic is saturating and every result
is clamped into the `u16` register width and to a floor of 1, so no source clock can produce a zero
count, an overflow, or a panic (`src/init/scl.rs:31`, `scl.rs:39`). This is exactly the "no SCL clock"
failure class that the debugging section covers: a controller whose count pairs were left at zero looks
alive but never toggles the line.

After bring-up the sequence acks the IRQ grant once (`src/setup/sequence.rs:16`) and assembles the
`Driver`.

### The transfer state machine (polled)

A transfer is bracketed by controller state changes and driven by a polling engine. `transfer`
validates the address and lengths, waits for the master to go idle, sets the target address, enables
the controller, runs the engine, and disables the controller again regardless of the engine's outcome
(`src/transaction/engine/transfer.rs:21`). `wait_idle` spins on `IC_STATUS` master-activity and returns
`Busy` if the master never goes idle (`src/transaction/control/wait_idle.rs:20`); `set_target`
disables the controller if needed, writes the 7-bit address to `IC_TAR`, and re-enables
(`src/transaction/control/set_target.rs:23`); `enable` and `disable` each wait for the enable-status bit
to reach the wanted state and time out otherwise
(`src/transaction/control/enable.rs:23`, `control/wait_enable_state.rs:20`).

The engine itself is `run` (`src/transaction/engine/run.rs:28`). It loops up to `TIMEOUT_ITERS`
(250,000) times over four phases:

1. **Abort check.** If `IC_RAW_INTR_STAT` shows the TX-abort bit, it latches `IC_TX_ABRT_SOURCE` into
   the result, reads `IC_CLR_TX_ABRT` to clear, and returns `Nack`
   (`src/transaction/engine/check_abort.rs:20`). This is the NACK path.
2. **Drain RX.** While the RX FIFO is not empty and read bytes remain, it pops `IC_DATA_CMD` low byte
   into the read buffer (`src/transaction/engine/drain_rx.rs:20`).
3. **Issue commands.** While commands remain and both the TX FIFO and RX FIFO have space, it issues one
   command word per iteration: a write byte for the write phase, or a read command (with a
   repeated-start on the first read if `FLAG_RESTART_ON_READ` is set) for the read phase, tagging the
   final command with `IC_DATA_CMD_STOP` (`run.rs:35`, `src/transaction/engine/read_cmd.rs:19`,
   `engine/take_write.rs:16`). FIFO space is computed against a fixed 64-entry depth
   (`engine/tx_space.rs:19`, `engine/rx_space.rs:19`).
4. **Completion.** Once every command has been issued and every expected read byte drained, `done`
   checks that the TX FIFO is empty and the master is no longer active, and returns the result
   (`run.rs:48`, `src/transaction/engine/done.rs:19`). If the budget runs out first, it returns
   `Timeout`.

This is the second honest gap and it is documented in the source itself. The engine is polled, not
interrupt driven: the IRQ is bound and acked once at bring-up so the interrupt line does not stay
asserted, but transfers busy-wait on FIFO status inside a fixed iteration budget rather than blocking on
the interrupt. The README lists interrupt-driven transfers as an explicit non-goal of this slice and
IRQ-aware completion as the named next step (`README.md:112`, `README.md:132`). The transfer buffers are
small and controller-local (64 bytes each), and every wait has a finite budget, so the driver fails
closed on a stuck bus rather than hanging.

## Protocol and IPC

The driver is one endpoint. Inbound, it serves the six opcodes above on the `driver.i2c_pci0` service
(port 4230) and replies on the reply endpoint (port 4231) through the app-independent IPC syscalls: it
receives with `mk_ipc_recv_from`, which yields the sender pid, and replies with `mk_ipc_reply` to that
pid (`src/server/runner.rs:19`, `src/server/respond.rs:6`). There is no other inbound surface.

The intended client is the i2c-HID driver. `capsule_driver_i2c_hid` resolves `driver.i2c_pci0` by name
through `mk_service_lookup` and issues `OP_TRANSFER` (opcode 5) requests, building the exact same `NI2C`
header (magic `0x4E49_3243`, version 1) and the same 8-byte transfer head, and setting
`FLAG_RESTART_ON_READ` when a write is followed by a read
(`userland/capsule_driver_i2c_hid/src/i2c_client/service.rs:4`,
`userland/capsule_driver_i2c_hid/src/i2c_client/wire.rs:1`, `wire.rs:7`). Its reply decoder reads the
same layout: status at offset 20, read length at 24, read bytes at 32
(`userland/capsule_driver_i2c_hid/src/i2c_client/wire.rs:28`). That is the whole runtime relationship:
HID-over-I2C sits above this driver and drives its descriptor and report registers through bounded
transfers.

Outbound, the only calls the capsule makes are the four broker syscalls that obtain and release its
hardware authority, all during bring-up:

```
  MkDeviceList     mk_device_list     enumerate PCI devices        src/discover.rs:36
  MkDeviceClaim    mk_device_claim    claim the controller         src/setup/claim.rs:3
  MkMmioMap        mk_mmio_map        map BAR0 (release on fail)    src/setup/mmio.rs:9
  MkIrqBind        mk_irq_bind        bind the INTx line            src/setup/irq.rs:7
```

The unwind paths call `mk_device_release` and `mk_mmio_unmap` to hand grants back on a failed step
(`src/setup/mmio.rs:11`, `src/setup/irq.rs:9`), and `mk_irq_ack` acks the bound vector once after
bring-up (`src/setup/sequence.rs:16`). The broker semantics behind these calls, the claim and epoch,
the BAR-bounded MMIO mapping with the MSI-X clamp, and the INTx bind, are documented on the
[device claim](../../subsystems/hardware-broker/claim.md),
[MMIO](../../subsystems/hardware-broker/mmio.md), and [IRQ](../../subsystems/hardware-broker/irq.md)
pages.

## Security analysis

This is a userspace driver, so the interesting question is how much hardware it can reach and what
bounds that reach. The answer is: exactly one Intel LPSS I2C controller, and only through broker grants
that the kernel can revoke.

The mask is six bits: `IPC`, `Memory`, `DeviceEnum`, `Driver`, `Mmio`, `Irq`
(`Capsule.mk:16`, `src/hardware/i2c_pci_capsule/spawn.rs:42`). There is no `DMA` bit, no `PIO` bit, no
filesystem, network, display, credential, or input-focus authority. So the driver cannot start a DMA
engine, cannot touch an I/O port, cannot read a file, cannot open a socket, and cannot see or steer
input. Its entire hardware footprint is the register window it maps and the interrupt it binds.

The broker enforces the bounds on that footprint, and the same three properties that hold for every
driver capsule hold here:

- **Exclusivity through the claim.** The driver must claim the controller before any grant is issued,
  and a device can be claimed by only one capsule at a time. Nothing else can be mapping this
  controller's BAR or taking its interrupt underneath it while the claim is held
  (`src/setup/claim.rs:3`; [device claim](../../subsystems/hardware-broker/claim.md)).
- **BAR-bounded MMIO with MSI-X withheld.** The mapping the driver receives is a slice of BAR0 the
  kernel resolved from its own device table, not an address the capsule named, and it is clamped to
  withhold any MSI-X table pages. So a bug in this capsule cannot walk the register base into another
  device's registers or into RAM, and cannot reprogram its own interrupt vectors
  (`src/setup/mmio.rs:9`; [MMIO](../../subsystems/hardware-broker/mmio.md)). The register accessor does
  raw volatile 32-bit reads and writes at fixed offsets into that granted window
  (`src/regs.rs:11`), so every access is inside the mapping the broker installed.
- **Kernel-owned interrupt vector.** The driver binds the INTx line but never programs the interrupt
  controller; the kernel owns the vector and the driver only acks
  (`src/setup/irq.rs:7`, `src/setup/sequence.rs:16`; [IRQ](../../subsystems/hardware-broker/irq.md)).

On revocation, the capsule unwinds its own grants on any failed bring-up step (release on a failed map,
unmap-and-release on a failed bind), and the broker releases every remaining grant a pid holds when the
process exits, so a crash cannot leak the controller (`src/setup/mmio.rs:11`, `src/setup/irq.rs:9`).

Input handling is defensive. The transfer address is checked against `0x7F` and the write/read lengths
against 64 before any register is touched (`src/transaction/engine/transfer.rs:25`), the protocol
decoder rejects malformed headers (`src/protocol/decode.rs:9`), and the transfer body parser rejects any
length that does not match its declared write length (`src/server/handlers/transfer.rs:30`). Every wait
loop has a finite iteration budget and fails closed with `Busy`, `Timeout`, or `Nack` rather than
spinning forever (`src/transaction/engine/run.rs:32`, `src/transaction/control/wait_idle.rs:21`). The
release profile is `panic = "abort"` and the capsule code carries no `unwrap`, `expect`, or `panic`;
errors are returned as status words (`Cargo.toml:18`).

Honest gaps, restated for the threat model. The polled transfer engine means a client can occupy the
driver for up to the full iteration budget per transfer; there is no preemption of an in-flight bus
transaction. Discovery is Intel and PCI only, so on a platform whose I2C controllers are ACPI-described
this driver simply does not bind anything, which is a coverage gap, not a safety one. Neither gap widens
the capsule's authority beyond the six-bit mask.

## How to contribute

The source lives at `userland/capsule_driver_i2c_pci/`. The tree is one unit per file:

- `src/main.rs` is the entry point; `src/setup/` is the bring-up sequence; `src/init/` is the
  controller reset and clock program.
- `src/discover.rs` and `src/constants/mod.rs` hold PCI discovery and the device/register tables.
- `src/protocol/` is the wire format (header, ops, errno, limits, decode, encode).
- `src/server/` is the receive loop and the per-op handlers under `src/server/handlers/`.
- `src/transaction/` is the transfer engine, split into `control/`, `engine/`, and `types/`.

To add support for another Intel LPSS controller, extend the PCI-id match in `device_info`
(`src/constants/mod.rs:28`) with the new id range and its source clock; discovery and bring-up pick it
up with no other change. To add an operation, define the opcode in `src/protocol/ops.rs`, re-export it
from `src/protocol/mod.rs`, add a handler module under `src/server/handlers/`, wire it into the match in
`src/server/runner.rs:35`, and keep fixed-width requests gated on an empty body the way the existing
four are.

To build and sign the capsule, use the generated per-slug make targets. The recipes are generated by
`nonos-mk/capsule.mk` from the `CAPSULE_*` variables in `Capsule.mk`, included at
`Makefile:661` (`nonos-mk/capsule.mk:158`, `Capsule.mk:19`):

```
  make nonos-mk-driver-i2c-pci               build the capsule ELF
  make nonos-mk-driver-i2c-pci-sign          produce the id cert, manifest, and attestation trailer
  make nonos-mk-driver-i2c-pci-verify        verify the signed artifacts against the trust anchor
  make nonos-mk-check-driver-i2c-pci-keys    check the per-capsule signing keys exist
```

The README also documents `make -B nonos-mk-driver-i2c-pci` as the build invocation
(`README.md:137`). For a kernel image that spawns the driver, `make nonos-mk-driver-i2c-pci-prod` builds
the profile with the `microkernel-driver-i2c-pci` feature and the driver's signed artifacts as
prerequisites (`Makefile:960`); the same artifacts are a prerequisite of the i2c-hid prod profile,
because the HID driver's client cannot run without the controller driver present (`Makefile:966`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every path returns an error as a status word, and the release profile is
`panic = "abort"`, `Cargo.toml:18`); modular files, one unit per file, with `mod.rs` used only for
re-exports (`src/transaction/engine/mod.rs:16`); and the AGPL header at the top of every source file,
matching the header already on the transaction and control modules
(`src/transaction/engine/run.rs:1`). The verification commands are a build, a kernel-profile check
`cargo check --no-default-features --features microkernel-driver-i2c-pci`, and the static gate `bash
nonos-ci/run-static-checks.sh` (`README.md:136`).

## Debugging

The first thing to confirm is that the capsule spawned. When the bus-driver spawn plan brings it up, the
boot log prints `[DRIVER-I2C-PCI] capsule spawned` on success and an `[ERROR]` line on a spawn failure
(`src/userspace/init/spawn_plan/drivers_bus.rs:41`, `src/userspace/init/capsule_boot/run.rs:29`,
`run.rs:32`). An absent line means the capsule never started, usually a signature, manifest, or
capability failure. The kernel-side load error carries the tag `[DRIVER-I2C-PCI] load_elf_executable
error:` (`src/hardware/i2c_pci_capsule/spawn.rs:48`).

Bring-up failure modes, each a distinct `Err(&str)` from the setup sequence:

- **Controller not found.** `find_controller` returned `None`, so `setup::run` fails with `i2c-pci:
  controller not found` and the capsule exits 1 (`src/setup/sequence.rs:10`, `src/main.rs:23`). The
  common cause is the Intel/PCI-only discovery gap: the controller is described in ACPI, not PCI, or its
  PCI id is not in `device_info`, or the record had no interrupt pin or a masked-off line
  (`src/discover.rs:41`). The broker device census (a `NONOS_DEVICE_CENSUS=1` build, documented on the
  [device claim](../../subsystems/hardware-broker/claim.md) page) shows whether the device is enumerated
  at all before any driver runs.
- **Claim / map / bind refused.** `device claim failed`, `mmio map failed`, or `irq bind failed` each
  name the broker step that was refused (`src/setup/claim.rs:6`, `src/setup/mmio.rs:12`,
  `src/setup/irq.rs:11`). The broker's own stage markers (for example the `[MMIO]` trace) isolate which
  check inside the grant failed; see the [MMIO](../../subsystems/hardware-broker/mmio.md) and
  [IRQ](../../subsystems/hardware-broker/irq.md) pages.
- **No SCL clock.** If the controller comes up but never disables cleanly, bring-up fails with `i2c-pci:
  controller disable timeout` (`src/init/mod.rs:55`). The subtler variant does not fail bring-up at all:
  a controller whose HCNT/LCNT pairs were never programmed emits no clock and every transfer NACKs or
  times out. `OP_TIMING_INFO` is the probe, it reads back the four SCL count registers, and a zero
  count there is the "no SCL clock" signature (`src/server/handlers/timing.rs:6`,
  `src/init/scl.rs:31`).

Runtime transfer failures map to the status words. `E_BUSY` means the master was still active when the
transfer began, a stuck or contended bus (`src/transaction/control/wait_idle.rs:27`). `E_TIMEOUT` means
the engine ran its full 250,000-iteration budget without completing, typically a missing clock, a device
that never fills the RX FIFO, or a controller that never reports done
(`src/transaction/engine/run.rs:54`). `E_NACK` means the DesignWare TX-abort bit fired; the abort source
is latched into the transfer reply so a client can read the exact abort reason
(`src/transaction/engine/check_abort.rs:24`, `src/server/handlers/transfer.rs:39`). `OP_PROBE` uses the
same path and distinguishes a clean NACK (`[0]`, absent) from a real error, so probing a known-good
address is the fastest way to tell a dead controller from an empty bus
(`src/transaction/engine/probe.rs:24`).

There is one more real-hardware trap worth naming, described on the
[IRQ](../../subsystems/hardware-broker/irq.md) page: a driver can claim its device and bind the IRQ, yet
no interrupt ever arrives because the IO-APIC redirect targets a LAPIC destination the running CPU is
not listening on. This driver polls its transfers, so it does not depend on interrupt delivery to make
progress, but the same routing gap explains why an interrupt-driven successor would stall where the
polled engine still completes.

## Source map

```
  src/main.rs                          _start -> setup::run -> server::run
  src/driver.rs                        Driver: device id, epoch, grants, clock, family, cached regs
  src/discover.rs                      MkDeviceList scan, Intel/PCI-only match
  src/constants/mod.rs                 PCI id -> family/clock table, DesignWare register offsets
  src/setup/                           bring-up sequence: claim, mmio, irq, sequence
  src/init/                            controller reset + SCL clock program (init/scl.rs)
  src/protocol/                        NI2C header, ops, errno, limits, decode, encode
  src/server/                          receive loop (runner.rs), respond.rs, handlers/
  src/server/handlers/                 health, controller, snapshot, timing, transfer, probe
  src/transaction/control/             wait_idle, set_target, enable, disable, wait_enable_state
  src/transaction/engine/              transfer, run (the polled state machine), probe, and helpers
  src/transaction/types/              TransferRequest/Result/Error, flags, valid_lengths
  Capsule.mk                           slug, handle, ports, capability mask, kernel mirror
  src/hardware/i2c_pci_capsule/        the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/drivers_bus.rs   the bus-driver spawn entry
  userland/capsule_driver_i2c_hid/src/i2c_client/   the i2c-hid client that drives OP_TRANSFER
  nonos-mk/capsule.mk                  the generated nonos-mk-driver-i2c-pci[-sign|-verify] targets
```

Every reference above is verified against those trees.
</content>
</invoke>

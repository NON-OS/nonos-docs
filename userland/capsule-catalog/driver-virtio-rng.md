# capsule_driver_virtio_rng

`capsule_driver_virtio_rng` is the first real userland hardware-driver capsule in the NONOS tree: a
signed user process that claims a virtio-rng PCI device through the kernel hardware broker, drives its
virtqueue over DMA, and serves raw entropy bytes to other capsules over IPC. It runs no crypto and keeps
no pool; it is the device-facing source that the entropy and crypto capsules sit above. The source is
`userland/capsule_driver_virtio_rng/`, and the kernel-side mirror that embeds, spawns, and calls it is
`src/hardware/virtio_rng_capsule/`.

## Contents

- [Overview and role](#overview-and-role)
- [Identity](#identity)
- [Operation reference](#operation-reference)
- [Architecture and bring-up](#architecture-and-bring-up)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview and role

The capsule owns one virtio-rng device end to end. At startup it enumerates the broker device table,
claims the device, maps its register window, binds its interrupt, allocates two DMA regions for the
virtqueue and the entropy buffer, brings the device through the virtio init handshake, and does a sanity
fill before it serves anything (`src/main.rs:40`, `src/setup/sequence.rs:26`). Once up, it answers two
IPC operations: fill a caller's buffer from device entropy, and a liveness probe. Everything the device
produces flows out over IPC; nothing crosses back in.

The capsule deliberately does no policy. It does not mix entropy, stretch it, run a CSPRNG, or make any
cryptographic decision; the README states that plainly, and the code holds no pool object at all
(`README.md:6`). Its relationship to [capsule_entropy](entropy.md) is a layering, not a duplication.
`capsule_entropy` is a monitored pass-through over the CPU `RDRAND` instruction executed in its own
context, and it holds no hardware capability and claims no device (see the entropy identity section,
which records that it carries only IPC, Memory, and Crypto). This driver is the opposite: it holds the
hardware authority and no crypto bit, and it draws bytes from a virtio device over a virtqueue rather
than from a CPU instruction. There is no in-tree wire between the two today: the only kernel caller of
this driver is the driver's own `CAP_DRIVER`-gated client, and a service that wanted to fold this source
into an entropy pool would layer its own check above that client
(`src/hardware/virtio_rng_capsule/capability.rs:18`). So the two are complementary entropy sources with
distinct authority, not a chain that is wired end to end in the shipping build.

The failure posture is fail-closed. Setup failure aborts startup and retries; a fill that the device
never completes returns an error; and there is no software fallback that fabricates entropy if the
hardware path cannot be established (`src/main.rs:43`, `README.md:69`).

## Identity

Everything the kernel and the service registry need to name and reach the capsule comes from its
`Capsule.mk` and the kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `driver-virtio-rng` | `Capsule.mk:6` |
| Service handle | `driver.virtio_rng` | `Capsule.mk:7`, `src/hardware/virtio_rng_capsule/spawn.rs:32` |
| Namespace | `systems.nonos.driver.virtio_rng` | `Capsule.mk:12` |
| Service endpoint | `service:4200:driver.virtio_rng` | `Capsule.mk:13`, `spawn.rs:33` |
| Reply endpoint | `reply:4201:endpoint.4294967302` | `Capsule.mk:14`, `spawn.rs:34` |
| Reply inbox name | `endpoint.4294967302` (= `0x1_0000_0006`) | `src/hardware/virtio_rng_capsule/client/transport.rs:27`, `src/protocol/endpoint.rs:21` |
| Capability mask | `0x1F8019` | `Capsule.mk:17` |
| Binary name | `driver_virtio_rng` | `Capsule.mk:10` |
| Kernel mirror | `src/hardware/virtio_rng_capsule` | `Capsule.mk:18` |

The capsule serves `driver.virtio_rng` on port 4200. The reply endpoint it sends responses to is the
kernel-owned inbox `endpoint.4294967302`, whose numeric form `0x1_0000_0006` is the constant
`KERNEL_REPLY_ENDPOINT` the capsule targets (`src/protocol/endpoint.rs:21`,
`src/server/error.rs:30`). That is slot 6 in the per-service reply-inbox numbering that runs
ramfs=1, keyring=2, entropy=3, crypto=4, vfs=5, virtio_rng=6, and the kernel mirror names the same slot
(`src/protocol/endpoint.rs:19`, `src/hardware/virtio_rng_capsule/client/transport.rs:25`). The reply port
itself is 4201 (`spawn.rs:34`).

The mask `0x1F8019` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x000001  CoreExec     bit()       1   types.rs:56
  0x000008  IPC          bit()       8   types.rs:59
  0x000010  Memory       bit()      16   types.rs:60
  0x008000  DeviceEnum   bit()   32768   types.rs:71
  0x010000  Driver       bit()   65536   types.rs:72
  0x020000  Mmio         bit()  131072   types.rs:73
  0x040000  Irq          bit()  262144   types.rs:74
  0x080000  Dma          bit()  524288   types.rs:75
  0x100000  Pio          bit() 1048576   types.rs:76
  --------
  0x1F8019  = 1 + 8 + 16 + 32768 + 65536 + 131072 + 262144 + 524288 + 1048576
```

There is a difference between the manifest mask and the runtime request worth stating plainly. The
`Capsule.mk` comment enumerates `IPC|Memory|Driver|DeviceEnum|Mmio|Irq|Dma|Pio` and writes that as
`0x1F8019` (`Capsule.mk:15`), but those eight bits sum to `0x1F8018`; the value `0x1F8019` carries one
extra bit, `CoreExec` (`0x1`), which the comment does not list. The kernel spawn path is the authority on
what the process actually receives, and it requests exactly `IPC | Memory | Driver | DeviceEnum | Mmio |
Irq | Dma | Pio` with no `CoreExec` bit (`spawn.rs:51`). So the value baked into the manifest is
`0x1F8019` (nine bits including `CoreExec`), while the caps the kernel requests at spawn are `0x1F8018`
(the eight the comment names). Either way there is no `Network` (4), no `FileSystem` (64), no `Crypto`
(32), no `Admin` (512), no `Debug` (256), and no graphics or input bit; the capsule can enumerate and
claim a device, map its registers by MMIO or PIO, bind its IRQ, allocate DMA, and speak IPC, and nothing
else.

## Operation reference

The wire frame is the 20-byte `NORD` header followed by an optional payload. The decoder validates that
the buffer carries a full header and that the magic and version match before anything dispatches; a bad
envelope returns `None` and the server answers `EINVAL` rather than acting on a stale protocol
(`src/protocol/decode.rs:26`, `src/server/runner.rs:44`). Two operations are defined
(`src/protocol/ops.rs:21`), routed in `src/server/runner.rs:51`; an unknown op returns `EINVAL`
(`src/server/runner.rs:54`).

| Op | Opcode | Handler | Request payload | Reply payload (after status) |
|---|---|---|---|---|
| `OP_FILL_RANDOM` | 1 | `handlers::fill::handle` | none; length is the header `payload_len` | up to `payload_len` entropy bytes |
| `OP_HEALTHCHECK` | 2 | `handlers::health::handle` | none | none |

Opcodes are `OP_FILL_RANDOM = 1` and `OP_HEALTHCHECK = 2` (`src/protocol/ops.rs:21`). Every reply carries
an `i32` status in the first four bytes of its payload, little-endian; a zero status means success
(`src/protocol/encode.rs:34`, `src/server/error.rs:29`).

### OP_FILL_RANDOM (op 1)

`fill::handle` (`src/server/handlers/fill.rs:31`) reads the requested length from the header's
`payload_len` field, not from a body word, and bounds it before touching the device:

```
  fill(driver, req):
      want = req.payload_len
      if want == 0 or want > MAX_FILL_BYTES:  EMSGSIZE       // 4096 ceiling
      n = fill(regs, queue, irq_grant)                       // one virtqueue round trip
      if n is Err:                            EIO            // device did not complete
      take = min(want, n)
      copy take bytes from the DMA buffer into the reply
      return status 0 + take bytes
```

The size ceiling is `MAX_FILL_BYTES = 4096` (`src/protocol/limits.rs:21`), which the limits comment ties
to the entropy buffer length so a single fill can never ask for more than the buffer holds
(`src/protocol/limits.rs:18`). The served count is the smaller of what the caller asked for and what the
device wrote into the used ring, so a short device write returns fewer bytes rather than padding
(`src/server/handlers/fill.rs:44`). The bytes are copied out of the capsule's own DMA grant into the
response buffer under a single-threaded server loop, so no concurrent device write is in flight while the
copy runs (`src/server/handlers/fill.rs:48`).

### OP_HEALTHCHECK (op 2)

`health::handle` (`src/server/handlers/health.rs:25`) reads no input and replies success with an empty
body. Reaching it proves the decoder accepted the envelope and the runner routed the op, so it is a
structural liveness probe; the kernel client uses it before a real fill so a partial bring-up (virtqueue
programmed but device idle) shows up as a distinct failure rather than a fill timeout
(`src/server/handlers/health.rs:17`).

### Errors

Three status codes are defined, mirroring Linux errnos so the kernel client can route them through the
same errno-to-error mapper it uses for the other capsules (`src/protocol/errno.rs:22`):

```
  E_INVAL    -22   malformed envelope, or an unknown op        errno.rs:22
  E_IO        -5   the device did not complete the fill        errno.rs:23
  E_MSGSIZE  -90   a fill request of zero or over 4096 bytes   errno.rs:24
```

A malformed envelope that fails to decode is answered `EINVAL` through a synthetic zero-valued request so
the caller is not left waiting on a reply that never comes (`src/server/error.rs:33`,
`src/server/runner.rs:47`).

## Architecture and bring-up

The capsule is `no_std`/`no_main`. `_start` initializes the heap, then loops on `setup::run` until the
whole broker chain succeeds, yielding 64 times between attempts on failure rather than spinning
(`src/main.rs:36`). On success it does a sanity fill and checks the result is not all zeros, releasing
every grant and exiting with code 4 if the device returned a dead buffer, or code 3 if the fill itself
failed (`src/main.rs:51`). Only then does it enter the server loop.

The eight top-level modules are `constants` (device ids, register offsets, queue layout, status bits),
`discover` (device-table match), `regs` (the MMIO-or-PIO register accessor), `setup` (the broker
bring-up chain), `queue` (the virtqueue), `init` (the virtio handshake), `fill` (one round trip), and
`server` (the IPC loop) (`src/main.rs:22`).

### Discovery

`find_virtio_rng` lists up to 32 device records through `mk_device_list` and returns the first that
matches (`src/discover/find.rs:23`). A match is a PCI device whose vendor is `0x1AF4` and whose device id
is either the transitional `0x1005` or the modern `0x1044` (`src/discover/is_match.rs:19`,
`src/constants/pci.rs:22`). The candidate must also advertise an interrupt pin and a usable line
(`irq_pin != 0`, `irq_line != 0xFF`) and expose at least one MMIO or PIO register BAR; the first such BAR
becomes the register window (`src/discover/find.rs:31`, `src/discover/first_register_bar.rs:18`). The
`Found` record carries the device id, IRQ line, and the register BAR index, kind, and size
(`src/discover/found.rs:16`).

### The broker chain

`setup::run` (`src/setup/sequence.rs:26`) runs the broker primitives in a fixed order, and each phase
rolls back every earlier grant in reverse order on failure so the broker never holds a partial setup:

1. **Claim.** `mk_device_claim` takes exclusive ownership of the device and returns the claim epoch that
   must travel with every later grant (`src/setup/claim.rs:24`). The broker refuses a device another
   capsule already holds and fences stale grants by epoch; the epoch check sits at the head of every
   grant path (see [device claim](../../subsystems/hardware-broker/claim.md)).
2. **Registers.** `registers::grant` maps the register window. If the discovered BAR is MMIO it calls
   `mk_mmio_map` for the page-rounded BAR length; if it is PIO it calls `mk_pio_grant`
   (`src/setup/registers/grant.rs:22`, `grant_mmio.rs:22`, `grant_pio.rs:20`). The resulting
   `RegisterGrant` hides the transport behind a uniform `Regs` accessor
   (`src/setup/registers/regs.rs:19`). The broker withholds a device's MSI-X table from any MMIO mapping,
   which is what keeps interrupt programming in the kernel (see
   [MMIO](../../subsystems/hardware-broker/mmio.md)).
3. **IRQ.** `irq::bind` binds the device interrupt. It tries legacy INTx first with the discovered line;
   on a platform where the line's GSI is not routed it falls back to MSI-X vector 1
   (`src/setup/irq.rs:28`). The broker leaves the source masked and the capsule unmasks it with
   `mk_irq_ack` once it is ready (`src/setup/irq.rs:20`, `src/setup/sequence.rs:50`). The capsule never
   touches the interrupt controller or the MSI-X table (see
   [IRQ](../../subsystems/hardware-broker/irq.md)).
4. **DMA.** Two grants. `dma::map_queue` allocates the two-page virtqueue region
   (`VQ_REGION_SIZE = 8192`), and `dma::map_buffer` allocates the one-page entropy buffer
   (`ENTROPY_BUF_LEN = 4096`) (`src/setup/dma.rs:30`, `src/constants/queue.rs:27`). Each returns both the
   user virtual address and the device-visible physical address; the frames are broker-allocated and
   zero-scrubbed before the capsule sees them, and the RNG device class ceiling is one page, which is
   exactly what the entropy buffer needs (see [DMA](../../subsystems/hardware-broker/dma.md)).
5. **Bring-up.** `bring_up` walks the legacy virtio status register: write 0, `ACKNOWLEDGE`, `DRIVER`,
   negotiate no feature bits (guest features cleared to keep the contract tight), `FEATURES_OK`, then
   select queue 0, read the max queue size, program the queue PFN as the physical page index, and finally
   `DRIVER_OK` (`src/init.rs:28`). It refuses to drive a device that advertises no requestq (queue max of
   zero) and writes `FAILED` if the features handshake is rejected (`src/init.rs:44`,
   `src/init.rs:51`). The status bits and their fixed write order come from the virtio 1.x spec
   (`src/constants/status.rs:22`, `src/constants/regs.rs:17`).

The live state is a `Driver` holding the device id, claim epoch, register grant, IRQ grant id, both DMA
grant ids, the `Queue`, and the `Regs` accessor (`src/setup/driver.rs:28`). Its `release` drops every
grant in reverse order, best-effort, so the kernel sees a clean teardown even on a voluntary exit
(`src/setup/driver.rs:44`).

### The virtqueue and how bytes are pulled

The queue is a single split virtqueue of size 16 (`src/constants/queue.rs:22`). The `Queue` struct is
`Copy` and carries the region and buffer pointers plus the `last_used` cursor and the computed available-
ring offset (`src/queue/layout.rs:23`). One descriptor at a time is enough: virtio-rng has one virtqueue,
and the device fills the buffer that descriptor 0 points at (`src/queue/post.rs:17`).

A fill is one round trip (`src/fill.rs:25`):

1. `post_request` writes descriptor 0 to point at the entropy buffer's physical address with the
   device-write flag set, publishes ring slot 0 in the available ring, and bumps the available `idx`
   after the slot write so the device never observes a partial ring update
   (`src/queue/post.rs:39`, `src/constants/queue.rs:23`).
2. The capsule pokes the legacy queue-notify register to tell the device to run
   (`src/fill.rs:28`, `src/constants/regs.rs:25`).
3. It waits for completion two ways at once: it polls the used ring's `idx` for the expected value, and
   it polls the IRQ grant's sequence counter through `mk_irq_poll` for a new interrupt, breaking on
   whichever fires first (`src/fill.rs:35`). A bounded `MAX_YIELDS` of 100,000 yields caps the wait and
   returns `"virtio-rng: device did not respond"` on timeout (`src/fill.rs:23`, `src/fill.rs:42`).
4. On completion it reads the byte count the device wrote into used-elem 0's `len` field, acknowledges
   the interrupt with `mk_irq_ack`, and returns the count (`src/fill.rs:48`, `src/queue/used.rs:42`).

The entropy bytes are then borrowed through `Queue::buffer`, which caps the slice at `buf_len` so a
misbehaving device cannot induce an out-of-bounds read (`src/queue/used.rs:55`). The register offsets for
the descriptor table, available ring, and used ring are computed from the legacy contiguous-region layout
with the used ring page-aligned at offset 4096 (`src/constants/queue.rs:25`, `src/queue/used.rs:27`).

## Protocol and IPC

The header is 20 bytes, little-endian, identical in shape to the entropy, crypto, vfs, ramfs, and keyring
capsules so one kernel-side client transport can serve them all uniformly (`src/protocol/header.rs:17`):

```
  u32 magic        0x4E4F5244 "NORD"    header.rs:30
  u16 version      1                    header.rs:31
  u16 op
  u16 flags
  u16 _reserved
  u32 request_id   echoed, not routed on
  u32 payload_len
```

The response reuses the same header, echoing the request's op, flags, and request id so the kernel client
can match replies, and prepends an `i32` status to its payload (`src/protocol/encode.rs:24`,
`src/protocol/encode.rs:34`). The server loop holds one request in flight at a time with a receive buffer
sized to just the header and a transmit buffer sized to the header plus the status word plus
`MAX_FILL_BYTES` (`src/server/runner.rs:33`).

The broker syscalls the capsule calls come from `nonos_libc` (`userland/libc/src/lib.rs:44`):

```
  mk_device_list    enumerate the device table            src/discover/find.rs:25
  mk_device_claim   claim the device, get the epoch       src/setup/claim.rs:25
  mk_mmio_map       map the register BAR (MMIO path)       src/setup/registers/grant_mmio.rs:25
  mk_pio_grant      grant the register ports (PIO path)    src/setup/registers/grant_pio.rs:22
  mk_pio_read/write drive the register window over PIO     src/regs/pio.rs:19, :25
  mk_irq_bind       bind INTx, falling back to MSI-X       src/setup/irq.rs:30, :34
  mk_irq_poll       read the interrupt sequence counter    src/fill.rs:56
  mk_irq_ack        unmask / acknowledge the interrupt     src/fill.rs:50, src/setup/sequence.rs:50
  mk_dma_map        allocate the queue and buffer regions  src/setup/dma.rs:37, :55
  mk_ipc_recv       receive a request on inbox 0           src/server/runner.rs:40
  mk_ipc_send       send the reply to the kernel inbox     src/server/error.rs:30, handlers/fill.rs:54
  mk_yield          yield while waiting                    src/fill.rs:45, src/main.rs:45
  mk_exit           abort on a dead source or failed fill  src/main.rs:37, :62, :67
```

Teardown uses the matching release calls, `mk_dma_unmap`, `mk_irq_unbind`, `mk_mmio_unmap` /
`mk_pio_release`, and `mk_device_release`, in reverse grant order (`src/setup/driver.rs:22`,
`src/setup/registers/release.rs:19`).

The one in-tree caller is the kernel-side client under `src/hardware/virtio_rng_capsule/client/`:

- `fill_random` (`src/hardware/virtio_rng_capsule/client/fill_random.rs:27`) gates on `CAP_DRIVER`
  through `gate_read`, refuses an over-4096 or empty request without round-tripping, writes the wanted
  length into the header's `payload_len` field, does a locked round trip, and copies the reply bytes back
  only if their count matches the request (`fill_random.rs:42`, `fill_random.rs:47`).
- `healthcheck` (`src/hardware/virtio_rng_capsule/client/healthcheck.rs:28`) gates on `CAP_DRIVER` and
  sends `OP_HEALTHCHECK`, returning `Ok` only when the round trip completes with status 0.

Both round-trip through the shared lifecycle transport to the kernel-owned reply inbox
`endpoint.4294967302` under a transport lock, mapping transport failures to typed `DriverRngError`
variants (`src/hardware/virtio_rng_capsule/client/transport.rs:37`).

## Security analysis

This is the most hardware-privileged capsule documented in this catalog, and its safety rests on the
broker's grant model rather than on the capsule being trusted. Its mask holds `DeviceEnum`, `Driver`,
`Mmio`, `Irq`, `Dma`, and `Pio` alongside `IPC` and `Memory` (`Capsule.mk:17`, `spawn.rs:51`), and it
holds no `Network`, `FileSystem`, `Crypto`, `Admin`, `Debug`, graphics, or input bit. It can drive its
one device and talk IPC; it cannot write a byte to disk, open a socket, or read another capsule's memory.

The entropy source is honest about what it is. The bytes are whatever the virtio-rng device writes into
the descriptor buffer, unmixed and unwhitened; the capsule does no conditioning and runs no health test
beyond checking that the sanity fill is not all zeros at startup (`src/main.rs:55`). It never fabricates
entropy: a fill the device does not complete returns `EIO`, and there is no software fallback path
(`src/server/handlers/fill.rs:39`, `README.md:69`). On QEMU the device is backed by the host RNG; on real
hardware the quality is exactly the device's, and this capsule adds nothing on top, which is why policy
belongs to the layers above it.

The hardware authority is contained by the broker, not by this capsule's good behavior:

- **Claim exclusivity and epoch.** The device is claimed exclusively, and the returned epoch travels with
  every grant; a release-and-reclaim invalidates any stale grant handle with `StaleEpoch`
  (`src/setup/claim.rs:24`, [claim](../../subsystems/hardware-broker/claim.md)).
- **Bounded registers.** The MMIO mapping can only name memory inside the BAR of the claimed device, and
  the broker withholds the MSI-X table from it, so this capsule cannot map arbitrary physical memory or
  program its own interrupt vectors ([MMIO](../../subsystems/hardware-broker/mmio.md)).
- **Kernel-owned interrupts.** The capsule programs no interrupt controller; it receives a grant id and a
  vector and drives the interrupt through `mk_irq_poll` and `mk_irq_ack`, so a compromised driver cannot
  redirect or forge an interrupt ([IRQ](../../subsystems/hardware-broker/irq.md)).
- **Bounded DMA.** The RNG device class ceiling is one page, so this capsule cannot drain physical memory
  through the DMA path, and its frames are zero-scrubbed before it or the device sees them
  ([DMA](../../subsystems/hardware-broker/dma.md)). The honest gap here is the broker's, not the
  capsule's: the IOMMU backend is not engaged in shipping builds, so the physical address the broker hands
  the device is a raw one and DMA safety rests on the software bounds plus non-malicious device hardware.
- **Fail-closed teardown.** Every grant is released in reverse order on a voluntary exit, and the kernel's
  `release_all_for_pid` on process exit guarantees a dying capsule leaks no claim, mapping, vector, or DMA
  region (`src/setup/driver.rs:44`, [revocation](../../subsystems/hardware-broker/revocation.md)).

The IPC surface is small and fail-closed. Both fills and health checks are bounded and validated, an
over-length or empty fill is refused with `EMSGSIZE` before the device is touched, and a malformed
envelope is answered `EINVAL` rather than dropped (`src/server/handlers/fill.rs:33`,
`src/server/runner.rs:47`). The device-written byte slice is capped at the buffer length so a lying used-
ring length cannot force an out-of-bounds read (`src/queue/used.rs:55`). The only kernel caller enforces
`CAP_DRIVER` before the IPC leaves the kernel, and the pid it checks comes from the kernel's process
accounting, not from any caller payload (`src/hardware/virtio_rng_capsule/capability.rs:25`).

Honest gaps: there is no conditioning or secondary source, so the output quality is exactly the device's;
the startup sanity check is a single all-zeros test, not a statistical health test, so a device that
returns biased-but-nonzero bytes passes it (`src/main.rs:55`); a fill that times out is an `EIO`, and the
completion wait is a bounded busy-poll of up to 100,000 yields rather than a true block on the interrupt
(`src/fill.rs:23`); and the manifest mask carries a `CoreExec` bit the `Capsule.mk` comment does not
enumerate, as noted in the identity section.

## How to contribute

The source lives at `userland/capsule_driver_virtio_rng/`. The wire format is under `src/protocol/`, the
IPC loop and handlers under `src/server/`, the broker bring-up chain under `src/setup/`, the virtqueue
under `src/queue/`, the register accessor under `src/regs/`, discovery under `src/discover/`, and the
device constants under `src/constants/`. Any change to the wire format or the op set has a kernel-side
counterpart under `src/hardware/virtio_rng_capsule/` that must stay in step.

To add or change an operation:

1. Add the opcode constant in `src/protocol/ops.rs:21`, and the matching constant in the kernel mirror
   under `src/hardware/virtio_rng_capsule/protocol/`.
2. Write the handler as one file under `src/server/handlers/`, exposing a `pub fn handle(...)` that
   funnels its error replies through `crate::server::error::reply_with_status` for a uniform response
   shape, and re-export it from `src/server/handlers/mod.rs:17`.
3. Wire it into the op match in `src/server/runner.rs:51`; an unrecognised op already returns `EINVAL`.
4. If the kernel needs to call it, add a client under `src/hardware/virtio_rng_capsule/client/`,
   re-export it from that module's `mod.rs`, and gate it through `gate_read`
   (`src/hardware/virtio_rng_capsule/capability.rs:25`) or a stricter check.

To build and sign the capsule, use the per-slug make targets generated from `Capsule.mk` through
`nonos-mk/capsule.mk` (`nonos-mk/capsule.mk:158`, included at `Makefile:655`):

```
  make nonos-mk-driver-virtio-rng              build the capsule ELF
  make nonos-mk-driver-virtio-rng-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-driver-virtio-rng-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-driver-virtio-rng-keys   check the per-capsule signing keys exist
```

The README's build line `make -B nonos-mk-driver-virtio-rng` forces a rebuild of that first target
(`README.md:129`). For a running kernel that embeds this driver, `make nonos-mk-driver-virtio-rng-prod`
builds the `microkernel-driver-virtio-rng` profile with the driver artifacts baked in (`Makefile:935`).
QEMU attaches the device with `-device virtio-rng-pci` (`Makefile:280`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every fallible path returns a `Result` or an error status, and the release
profile is `panic = "abort"`, `Cargo.toml:25`); modular files, one unit per file, with `mod.rs` used only
for re-exports (`src/protocol/mod.rs`, `src/setup/mod.rs`, `src/server/mod.rs`); and the AGPL header at
the top of every source file, matching the header on every existing module (`src/main.rs:1`).

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[DRIVER-VIRTIO-RNG] capsule spawned` from the boot log (tag `DRIVER-VIRTIO-RNG`, message `capsule
spawned`), driven by the shared capsule-boot helper (`src/userspace/init/spawn_plan/drivers_virtio_io.rs:26`,
`src/userspace/init/capsule_boot/run.rs:29`, format at `src/sys/boot_log/output.rs:33`). An absent line
means the capsule never started, usually a signature, manifest, or capability failure, and the error path
prints an `[ERROR]` line instead (`src/userspace/init/capsule_boot/run.rs:32`). The spawn also carries a
debug tag `[DRIVER-VIRTIO-RNG] load_elf_executable error:` for an ELF load failure
(`src/hardware/virtio_rng_capsule/spawn.rs:59`).

Failure modes and where to look:

- The capsule spawns but immediately exits. `_start` exits with a specific code for each early failure:
  code 1 if the heap will not initialize, code 3 if the sanity fill failed, and code 4 if the device
  returned an all-zero buffer (`src/main.rs:37`, `src/main.rs:62`, `src/main.rs:67`). A code-4 exit is the
  distinctive "device present but producing nothing" signature.
- Setup never completes, so the server never starts. `setup::run` retries forever, yielding 64 times
  between attempts (`src/main.rs:40`). The blocking stage is one of the broker phases, and each one prints
  its own stage markers on the console: an `AlreadyClaimed` or `UnknownDevice` at claim
  ([claim](../../subsystems/hardware-broker/claim.md)), an `[MMIO]` stage trace that stops before
  `[MMIO] record` ([MMIO](../../subsystems/hardware-broker/mmio.md)), an `IrqBindError`
  ([IRQ](../../subsystems/hardware-broker/irq.md)), or a `[DMA]` stage line
  ([DMA](../../subsystems/hardware-broker/dma.md)). A `NONOS_DEVICE_CENSUS=1` build renders the broker
  device table so you can confirm the virtio-rng device is even present before any driver runs.
- The device is discovered but refused. Discovery skips a match with no interrupt pin or a masked line,
  and refuses a device with no register BAR (`src/discover/find.rs:31`); `bring_up` refuses a device
  advertising no requestq (`src/init.rs:51`). Either surfaces as a setup failure and a retry, not a
  crash.
- A fill returns `EIO`. The device did not post the used ring within `MAX_YIELDS` (100,000) yields
  (`src/fill.rs:42`), so the handler answered `EIO` (`src/server/handlers/fill.rs:39`). On real hardware
  the usual cause is that the interrupt never fired, which the [IRQ](../../subsystems/hardware-broker/irq.md)
  debugging section covers (IO-APIC destination, bus-master or INTx disabled, an MSI-X enable bit never
  set); the fill also polls the used-ring `idx` directly, so a genuine `EIO` means neither the interrupt
  nor the ring advanced.
- A fill returns `EMSGSIZE`. The request asked for zero bytes or more than 4096
  (`src/server/handlers/fill.rs:33`); the kernel client also rejects an over-4096 or empty request before
  it ever round-trips (`src/hardware/virtio_rng_capsule/client/fill_random.rs:29`).
- Any op returns `EINVAL`. A malformed envelope (bad magic, wrong version, short buffer) or an unknown op
  (`src/server/runner.rs:47`, `src/server/runner.rs:54`). This is distinct from the `EIO` that means the
  device itself did not complete.
- The health check is the isolation tool. `OP_HEALTHCHECK` proves the capsule is alive and its decoder
  and router work without consuming entropy or touching the device path, so a healthy probe with failing
  fills points at the virtqueue or interrupt path rather than at IPC
  (`src/server/handlers/health.rs:25`, `src/hardware/virtio_rng_capsule/client/healthcheck.rs:28`).

## Source map

```
  userland/capsule_driver_virtio_rng/src/main.rs             _start: heap, setup retry, sanity fill, server
  userland/capsule_driver_virtio_rng/src/discover/           device-table match (vendor/device, IRQ, BAR)
  userland/capsule_driver_virtio_rng/src/setup/sequence.rs   the broker bring-up chain in order
  userland/capsule_driver_virtio_rng/src/setup/{claim,irq,dma}.rs   the per-phase broker calls + rollback
  userland/capsule_driver_virtio_rng/src/setup/registers/    MMIO-or-PIO register grant and release
  userland/capsule_driver_virtio_rng/src/setup/driver.rs     Driver: the live grants + reverse-order release
  userland/capsule_driver_virtio_rng/src/init.rs             the virtio legacy init handshake
  userland/capsule_driver_virtio_rng/src/queue/              the split virtqueue (post_request, used ring)
  userland/capsule_driver_virtio_rng/src/fill.rs             one virtqueue round trip + completion wait
  userland/capsule_driver_virtio_rng/src/regs/               the MMIO/PIO register accessor
  userland/capsule_driver_virtio_rng/src/constants/          device ids, register offsets, queue layout, status
  userland/capsule_driver_virtio_rng/src/protocol/           the NORD frame, ops, errno, limits, endpoint
  userland/capsule_driver_virtio_rng/src/server/             the IPC loop, error path, fill/health handlers
  userland/capsule_driver_virtio_rng/Capsule.mk              slug, handle, ports, capability mask, mirror
  src/hardware/virtio_rng_capsule/spawn.rs                   the verified kernel-side spawn + requested caps
  src/hardware/virtio_rng_capsule/protocol/                  the kernel mirror of the wire format
  src/hardware/virtio_rng_capsule/client/                    the CAP_DRIVER-gated kernel client (fill/health)
  src/hardware/virtio_rng_capsule/capability.rs              the CAP_DRIVER read gate
  src/userspace/init/spawn_plan/drivers_virtio_io.rs         spawn_rng at boot
  src/capabilities/types.rs                                  the capability bit values
```

Every reference above is verified against those trees.
</content>

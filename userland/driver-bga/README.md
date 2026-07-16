# capsule_driver_bga (full reference)

`capsule_driver_bga` is the Bochs Graphics Adapter (BGA) display capsule in the NONOS tree: a userland
capsule that claims the QEMU/Bochs standard-VGA PCI device, sets a linear-framebuffer mode through the
VBE DISPI register interface, and paints a solid clear colour into the framebuffer. It is a
scanout-provider peer to the GOP and virtio-gpu paths, and it is the simplest of the three: one PCI
device, two MMIO BARs, four register writes to set a mode, and a linear 32-bit framebuffer. It reaches
hardware exclusively through the [hardware broker](../../subsystems/hardware-broker/README.md), never
through kernel driver code (`userland/capsule_driver_bga/Cargo.toml:5`).

Be honest about its status. This capsule is not a primary display backend today, and it is not a
promoted, brokered, always-on driver either. It has no `Capsule.mk`, so it has no service handle, no
service or reply port, no capability mask, and no entry in the build-and-sign system; its own README
calls it a parked source inventory for a future brokered BGA display capsule
(`userland/capsule_driver_bga/README.md:5`, `README.md:36`). What the source does do, end to end, is a
real one-shot bring-up: discover, claim, bus-master, map both BARs, set the mode, clear the screen, and
then park in a yield loop (`src/main.rs:33`). This page documents exactly that, and marks every gap
between the code and a production driver where it exists.

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

The capsule is `no_std`/`no_main`. `_start` runs the one-shot setup sequence and, on success, keeps the
resulting `Driver` alive and parks the capsule in a `mk_yield` loop; on failure it exits with a code
derived from the error (`src/main.rs:33`). Holding the `Driver` alive is what keeps the display up: the
`Driver` owns the broker grants, and dropping it would unmap the framebuffer and release the device, so
the yield loop is load-bearing, not idle (`src/setup/driver.rs:19`, `src/handles.rs:31`).

Setup is the whole capsule. `setup::run` discovers the BGA PCI function, claims it, enables PCI bus
mastering, maps the register BAR and then the framebuffer BAR, sets a 1024x768x32 linear-framebuffer
mode through the DISPI registers, clears the framebuffer to a solid colour, and returns a `Driver` that
records the framebuffer virtual address, the resolution, and the stride (`src/setup/sequence.rs:27`).
There is no request loop after that, because there is no service endpoint: unlike the storage and input
driver capsules, this one does not serve clients over IPC. It brings the panel up once and then parks.

The mode is fixed. Width, height, and bit depth are compile-time constants (1024x768, 32bpp), not
parameters a client selects, and the clear colour is a constant as well (`src/constants.rs:33`). This is
the honest shape of a bring-up capsule: it proves the DISPI path and the two BAR mappings on real or
emulated hardware, and a promoted version would add the client protocol and the mode negotiation that
are absent today.

## Identity

There is no `Capsule.mk` in this capsule, so the identity fields that name and reach a production driver
do not exist yet. What identity the capsule has comes from its `Cargo.toml` and its source constants.

| Field | Value | Source |
|---|---|---|
| Crate name | `nonos_capsule_driver_bga` | `Cargo.toml:11` |
| Binary name | `driver_bga` | `Cargo.toml:20` |
| Capsule slug | none (no `Capsule.mk`) | `userland/capsule_driver_bga/` has no `Capsule.mk` |
| Service handle | none while parked | `README.md:15` |
| Service endpoint | none while parked | `README.md:15` |
| Reply endpoint | none while parked | `README.md:15` |
| Capability mask | not declared (no `CAPSULE_REQUIRED_CAPS`) | `README.md:9`, `README.md:63` |
| Kernel mirror | none | no `src/*/bga*` tree exists |
| PCI identity it matches | vendor `0x1234`, device `0x1111`, class `0x03` | `src/constants.rs:17` |

Because there is no manifest, there is no capability mask to decompose. What a promoted version would
need is spelled out by its sibling, the virtio-gpu driver, whose manifest declares
`CAPSULE_REQUIRED_CAPS := 0x1F9019` for `CoreExec | IPC | Memory | GraphicsSurfaceCreate | DeviceEnum |
Driver | Mmio | Irq | Dma | Pio` (`userland/capsule_driver_virtio_gpu/Capsule.mk:11`). The BGA capsule
uses a strict subset of the broker syscalls (no IRQ, no DMA, no PIO, no surface), so the capabilities its
code actually exercises are exactly these five, each decomposed against `src/capabilities/types.rs`:

```
  0x00001  CoreExec       bit()      1     types.rs:56    the capsule runs at all
  0x08000  DeviceEnum     bit()  32768     types.rs:71    mk_device_list          (discover.rs:34)
  0x10000  Driver         bit()  65536     types.rs:72    mk_device_claim/release, mk_pci_config_write
  0x20000  Mmio           bit() 131072     types.rs:73    mk_mmio_map / mk_mmio_unmap
  ------
  (IPC 0x8 and Memory 0x10 would also be needed once it serves clients and owns a heap)
```

The `DeviceEnum` / `Driver` / `Mmio` split is the broker's own vocabulary: `DeviceEnum` is
enumerate-only, `Driver` lets a capsule claim and release a device, and `Mmio` lets a claim holder map a
BAR slice into its own address space (`src/capabilities/types.rs:35`). The BGA source touches no IRQ, DMA,
or PIO wrapper, so a promoted manifest would not carry `Irq` (0x40000), `Dma` (0x80000), or `Pio`
(0x100000).

## Operation reference

This capsule exposes no client operations. It has no opcodes, no request or reply layout, and no wire
format, because it registers no service and runs no request loop (`src/main.rs:33`; there is no `server`
module and no IPC call anywhere in `src/`). "Operations" here means the ordered privileged steps the
one-shot bring-up performs, each a broker syscall or a DISPI register write, and each is the real unit of
work the capsule does.

Broker operations, in the order `setup::run` performs them (`src/setup/sequence.rs:27`):

| Step | Broker call | Purpose | Source |
|---|---|---|---|
| discover | `mk_device_list(CLASS_DISPLAY, ...)` | enumerate display-class devices, match the BGA | `discover.rs:34`, `discover.rs:41` |
| claim | `mk_device_claim(device_id)` | take exclusive ownership, receive the epoch | `setup/claim.rs:22` |
| bus-master | `mk_pci_config_write(COMMAND, BUS_MASTER)` | set the PCI Command Bus Master Enable bit | `setup/pci.rs:22` |
| map regs | `mk_mmio_map(dev, epoch, REG_BAR=2, ...)` | map the DISPI register BAR into the capsule | `setup/mmio.rs:23` |
| map fb | `mk_mmio_map(dev, epoch, FB_BAR=0, ...)` | map the linear framebuffer BAR into the capsule | `setup/mmio.rs:23` |
| release regs/fb | `mk_mmio_unmap(grant_id)` (x2) | drop both mappings on teardown | `handles.rs:33` |
| release device | `mk_device_release(device_id)` | drop the claim on teardown | `handles.rs:35` |

DISPI register operations, performed by `set_mode` against the mapped register BAR
(`src/dispi/set_mode.rs:24`), each a 16-bit MMIO write at offset `0x500 + index*2`
(`src/dispi/dispi_off.rs:19`):

| Write | Register (index) | Value | Source |
|---|---|---|---|
| disable | ENABLE (4) | `0` | `set_mode.rs:26` |
| x resolution | XRES (1) | `width` (1024) | `set_mode.rs:27` |
| y resolution | YRES (2) | `height` (768) | `set_mode.rs:28` |
| bit depth | BPP (3) | `32` | `set_mode.rs:29` |
| enable | ENABLE (4) | `DISPI_ENABLED \| DISPI_LFB_ENABLED` (0x41) | `set_mode.rs:30` |

Framebuffer operation, performed by `clear` after the mode is set: it writes a solid 32-bit colour into
every pixel of the mapped framebuffer, `width * height` pixels, one volatile `u32` store per pixel
(`src/dispi/clear.rs:19`, `sequence.rs:38`).

Errors are a two-variant enum, mapped to process exit codes since there is no reply channel to carry a
status (`src/error.rs:17`):

| Error | When | Exit code | Source |
|---|---|---|---|
| `DeviceNotFound` | no display device matched the BGA identity | `2` | `error.rs:27`, `sequence.rs:28` |
| `BrokerCallFailed(rc)` | any broker syscall returned a negative rc | `3` | `error.rs:28`, `setup/claim.rs:23` |

A `BrokerCallFailed` carries the raw negative broker return code (`error.rs:20`), though `exit_code`
collapses both cases to a fixed number, so the specific rc is visible only if the setup call is traced,
not in the exit status (`error.rs:25`).

## Architecture and bring-up

The bring-up is one linear function, `setup::run`, with each step in its own module. On any error after
the claim it releases what it took, so a partial bring-up never leaves the device claimed or a BAR
mapped (`src/setup/sequence.rs:30`, `setup/mmio.rs:25`).

1. **Discover.** `find_bga` calls `mk_device_list` for the display class into a fixed 32-entry buffer and
   scans the returned records for a PCI device with class `0x03`, vendor `0x1234`, device `0x1111`, at
   least three BARs, and both BAR 0 (framebuffer) and BAR 2 (registers) present as MMIO BARs with
   non-zero size (`src/discover.rs:32`, `discover.rs:41`). It returns the broker device id and the two
   BAR sizes, or `None` if nothing matches, which becomes `DeviceNotFound`
   (`discover.rs:51`, `sequence.rs:28`).
2. **Claim.** `claim` calls `mk_device_claim` and keeps the returned epoch (`src/setup/claim.rs:22`). The
   claim is exclusive and the epoch is the token every later grant is checked against
   ([device claim](../../subsystems/hardware-broker/claim.md)).
3. **Bus-master.** `enable_bus_master` writes the PCI Command register's Bus Master Enable bit through
   `mk_pci_config_write` with `MK_PCI_CFG_COMMAND` (0x04) and `MK_PCI_CMD_BUS_MASTER` (1<<2)
   (`src/setup/pci.rs:22`, `userland/libc/src/broker/pci.rs:26`). The broker only accepts a write into
   that specific register-and-bit, so this is the one PCI configuration change the capsule can make
   (`userland/libc/src/broker/pci.rs:17`). If this fails, setup releases the device and returns
   (`sequence.rs:30`).
4. **Map the register BAR.** `mmio::map` calls `mk_mmio_map` for BAR 2 over the whole reported BAR size,
   receiving a user virtual address and a grant id in an `MmioMapOut`
   (`src/setup/mmio.rs:23`, `userland/libc/src/broker/types/mmio.rs:19`). The broker maps the slice
   uncached, no-execute, read-write, bounded to the BAR, and stops short of any MSI-X table
   ([MMIO grants](../../subsystems/hardware-broker/mmio.md)). This VA becomes the `Regs` register window
   (`src/regs.rs:24`, `sequence.rs:36`).
5. **Map the framebuffer BAR.** The same `mmio::map` runs for BAR 0, giving the linear framebuffer
   virtual address the capsule writes pixels into (`src/setup/mmio.rs:23`, `sequence.rs:35`).
6. **Set the mode.** `set_mode` writes the DISPI registers over the mapped register window. The DISPI
   interface is index-addressed 16-bit registers at MMIO offset `0x500` inside the register BAR, one
   register every two bytes (`src/constants.rs:24`, `src/dispi/dispi_off.rs:19`). The sequence disables
   the adapter, programs XRES, YRES, and BPP=32, then re-enables with both the enable bit and the
   linear-framebuffer bit set (`0x01 | 0x40`), which is the standard BGA mode-set protocol: you must
   clear enable before changing resolution and set it last (`src/dispi/set_mode.rs:24`,
   `src/constants.rs:29`).
7. **Clear.** `clear` fills `1024 * 768` pixels of the framebuffer with the constant colour `0x00102A3A`,
   one volatile 32-bit store per pixel, so a successful bring-up shows a solid dark-teal panel rather
   than uninitialised memory (`src/dispi/clear.rs:19`, `src/constants.rs:36`, `sequence.rs:38`).
8. **Hand back the `Driver`.** `run` builds the `Driver` holding the `BrokerHandles` (device id and the
   two grant ids), the framebuffer VA, the width, the height, and the stride
   (`width * 4 = 4096` bytes) (`src/setup/sequence.rs:40`, `src/setup/driver.rs:19`). `_start` keeps it
   alive in the yield loop; if it were dropped, `BrokerHandles::drop` would unmap both BARs and release
   the device, tearing the display down (`src/handles.rs:31`).

The register access is deliberately thin. `Regs` is a copy of the base VA with `r16`/`w16` volatile
accessors, and `dispi_off` turns a DISPI index into the byte offset; there is no abstraction beyond the
two accessors and the offset helper (`src/regs.rs:29`, `src/dispi/dispi_off.rs:19`). Note that despite
the constant name `DISPI_IOPORT_OFFSET`, this capsule reaches the DISPI registers through the MMIO
register BAR, not through x86 I/O ports: it never mints a PIO grant and never issues `in`/`out`
(`src/constants.rs:24`, `src/setup/mmio.rs:23`).

## Protocol and IPC

There is no protocol and no IPC. This capsule speaks only the broker syscall ABI, and it registers no
service, accepts no requests, and sends no replies. The only "wire" it touches is the six-word broker
syscall envelope, `call_raw(number, [args; 6])` (`userland/libc/src/syscall/bridge.rs`, re-exported at
`userland/libc/src/syscall/mod.rs:22`). The broker syscalls it invokes, with their tag-encoded numbers:

```
  mk_device_list      MDLS   enumerate a device class     broker/device.rs:24, syscall/numbers/broker.rs:18
  mk_device_claim     MDCL   claim a device, get epoch    broker/device.rs:29, syscall/numbers/broker.rs:19
  mk_device_release   MDRL   release a claim              broker/device.rs:34, syscall/numbers/broker.rs:20
  mk_pci_config_write MPCW   set the Bus Master bit       broker/pci.rs:42,    syscall/numbers/broker.rs:36
  mk_mmio_map         MMMP   map a BAR slice              broker/mmio.rs:25,   syscall/numbers/broker.rs:21
  mk_mmio_unmap       MMUM   unmap a BAR slice            broker/mmio.rs:39,   syscall/numbers/broker.rs:22
```

`mk_mmio_map` packs the BAR index and flags into a single argument register so the call stays inside the
six-word envelope; the BGA capsule passes flags `0` and offset `0`, mapping each BAR from its base
(`userland/libc/src/broker/mmio.rs:34`, `src/setup/mmio.rs:23`). `mk_device_list` writes an array of
`DeviceRecord` (176 bytes each) into the capsule's buffer and returns the count
(`userland/libc/src/broker/types/device.rs:24`, `discover.rs:34`); each record carries the PCI identity
and six `Bar` entries the discover filter reads (`userland/libc/src/broker/types/bar.rs:23`).

A promoted version would register a display service and add exactly the IPC surface the README lists as
missing: a stable service endpoint, a client protocol, and a manifest that declares the capabilities
(`README.md:9`, `README.md:15`, `README.md:60`). None of that exists in the current source.

## Security analysis

The BGA capsule is a userland driver with no ambient authority. Everything it can touch, it touches
through a broker grant that the kernel checks against a claim it holds, and it holds nothing beyond one
display device and two mappings of that device's own BARs.

The claim is exclusive and epoch-stamped. `mk_device_claim` refuses a device another capsule already
holds, and the epoch it returns is quoted on every later grant and re-checked, so a stale grant from a
prior ownership cannot be replayed after the device changes hands
([device claim](../../subsystems/hardware-broker/claim.md)). The BGA capsule threads that epoch through
the bus-master write and both MMIO maps (`src/setup/pci.rs:22`, `src/setup/mmio.rs:23`), so all three
grants are bound to the same claim.

The MMIO mappings are bounded and safe by construction. The broker computes the physical range from the
kernel's device table, not from the request, checks it lies inside the BAR, maps it uncached,
no-execute, read-write, adds a guard page between grants, and withholds any MSI-X table page
([MMIO grants](../../subsystems/hardware-broker/mmio.md)). So even though the capsule writes raw 16-bit
values into the register BAR and raw 32-bit values into the framebuffer BAR, it can only reach memory
inside two BARs of a device it claims; a bug in the mode-set sequence or the clear loop cannot walk into
another device's registers or into RAM, and the no-execute attribute means the writable framebuffer is
not a code-injection path.

The PCI configuration authority is a single bit. The broker only accepts a `mk_pci_config_write` into
the Command register's Bus Master Enable bit (or the MSI-X control bits, which this capsule never
touches); every other offset and bit pattern is rejected (`userland/libc/src/broker/pci.rs:17`). So the
capsule cannot reprogram BARs, move the device, or change its command word beyond enabling bus mastering.

Teardown is complete and automatic. `BrokerHandles::drop` unmaps the framebuffer, unmaps the registers,
and releases the device, in that order, and the broker's `release_all_for_pid` runs from the process
exit path as a backstop, so a crash cannot leak the claim or the mappings
(`src/handles.rs:31`, [device claim](../../subsystems/hardware-broker/claim.md)). Isolation from other
capsules is the kernel's: this is a CPL 3 user binary that only makes broker syscalls, and once promoted
it would be verified and enrolled at spawn like every other capsule.

The honest security caveat is the same as its status caveat. Because there is no manifest, the capsule
is not enrolled and not spawned in any production profile today, so this analysis describes the authority
the code would exercise once promoted, bounded by the broker, not authority it is granted on a shipping
image (`README.md:17`, `README.md:27`).

## How to contribute

The source lives at `userland/capsule_driver_bga/`. It is small and layered: `src/main.rs` is the entry
and park loop, `src/setup/` is the one-shot bring-up (one module per step: `claim`, `pci`, `mmio`, plus
`sequence` that orders them and `driver` that holds the result), `src/dispi/` is the mode-set, clear, and
register-offset helpers, `src/discover.rs` is the PCI match, `src/regs.rs` is the volatile register
accessor, `src/handles.rs` is the RAII broker teardown, `src/error.rs` is the error type, and
`src/constants.rs` is every device id, register index, and mode constant.

To promote it to a real display capsule, follow the checklist the README already states
(`README.md:60`):

1. Add a `Capsule.mk` declaring the slug, handle, namespace, service and reply endpoints, binary name,
   kernel mirror, and `CAPSULE_REQUIRED_CAPS`, then `include nonos-mk/capsule.mk`, exactly as the
   virtio-gpu driver does (`userland/capsule_driver_virtio_gpu/Capsule.mk:1`). The generated per-slug
   targets come from `nonos-mk/capsule.mk:158` and give you build, sign, verify, and key-check targets.
2. Set `CAPSULE_REQUIRED_CAPS` to the mask the code exercises. The five capabilities are `CoreExec`,
   `DeviceEnum`, `Driver`, `Mmio`, and, once it serves clients, `IPC` and `Memory`
   (`src/capabilities/types.rs:56`); do not add `Irq`, `Dma`, or `Pio`, which the source does not use.
3. Register a signed spawn path (a kernel mirror and a spawn-plan entry) and regenerate the trust
   artifacts, as the checklist requires (`README.md:63`).
4. Add the client protocol and mode negotiation that are missing today, so the fixed 1024x768x32 mode
   (`src/constants.rs:33`) becomes a request rather than a constant.

There are no `make` targets for this capsule today, because no `Capsule.mk` includes
`nonos-mk/capsule.mk`; a grep of the Makefile and `nonos-mk/` finds no `driver-bga` or `driver_bga`
slug (verified against `Makefile` and `nonos-mk/capsule.mk`). Adding the `Capsule.mk` in step 1 is what
mints `nonos-mk-driver-bga`, `nonos-mk-driver-bga-sign`, `nonos-mk-driver-bga-verify`, and
`nonos-mk-check-driver-bga-keys` (the pattern at `nonos-mk/capsule.mk:158`). The only build the source
supports today is a direct `cargo build` of the crate.

Code standards the capsule already meets and a change must keep: `cargo fmt` and a clean `cargo clippy`;
no panics, `unwrap`, or `expect` (setup returns a `BgaResult` and the entry maps the error to an exit
code, and the release profile is `panic = "abort"`, `Cargo.toml:27`); modular files, one unit per file,
with `mod.rs` used only for re-exports (`src/setup/mod.rs:23`, `src/dispi/mod.rs:21`); and the AGPL
header at the top of every source file (`src/main.rs:1`).

## Debugging

The capsule has no log markers of its own; it prints nothing, because it registers no service and calls
no debug syscall. What it produces instead is a process exit code on failure and a solid dark-teal panel
on success. Diagnose it through the broker's own traces and the exit code.

Failure modes and where to look:

- **Nothing on screen, capsule exits with code 2.** `find_bga` matched no device, so setup returned
  `DeviceNotFound` (`src/error.rs:27`, `sequence.rs:28`). Either the QEMU stdvga device is absent, or its
  PCI identity or BAR layout does not match the filter: it requires vendor `0x1234`, device `0x1111`,
  class `0x03`, at least three BARs, and both BAR 0 and BAR 2 as non-zero MMIO BARs
  (`src/discover.rs:41`). The broker device census, a `NONOS_DEVICE_CENSUS=1` build, renders the broker
  device table to the framebuffer so you can confirm the device is enumerated at all before this capsule
  runs ([device claim](../../subsystems/hardware-broker/claim.md)).
- **Capsule exits with code 3.** A broker call returned a negative rc, wrapped as
  `BrokerCallFailed(rc)` (`src/error.rs:28`). The failing call is one of claim, bus-master, or either
  MMIO map, in that order (`sequence.rs:29`). A claim rejection is usually `AlreadyClaimed` (another
  capsule holds the display, for example the virtio-gpu driver) or a stale epoch; a map rejection prints
  the broker's own `[MMIO]` stage markers on a `NONOS_FBCONSOLE=1` build, and the missing marker names
  the step that blocked it ([MMIO grants](../../subsystems/hardware-broker/mmio.md)). The raw rc is
  inside the error value but not in the exit code, which is fixed at 3, so trace the setup call to see it
  (`src/error.rs:25`).
- **Panel comes up but at the wrong resolution.** The mode is the compile-time constant 1024x768x32
  (`src/constants.rs:33`); this capsule does not read EDID and does not negotiate a mode, so a display
  that does not accept 1024x768 over the DISPI linear-framebuffer path shows whatever the firmware left.
  Changing the mode today means changing the constants and rebuilding.
- **Panel is garbled or a partial fill.** The clear loop writes exactly `width * height` pixels from the
  framebuffer base with a stride assumed equal to `width * 4` (`src/dispi/clear.rs:19`,
  `sequence.rs:41`). If the adapter's real scanline stride differs from `width * 4`, or the framebuffer
  BAR is smaller than `width * height * 4`, the fill will not line up; the capsule does not read back a
  hardware stride register, it assumes the packed 32-bit layout the DISPI mode-set requests.

## Source map

```
  src/main.rs                    _start -> setup::run, then park in mk_yield
  src/setup/sequence.rs          the one-shot bring-up (discover, claim, bus-master, map x2, set mode, clear)
  src/setup/claim.rs             mk_device_claim, returns the epoch
  src/setup/pci.rs               mk_pci_config_write: Bus Master Enable
  src/setup/mmio.rs              mk_mmio_map for a BAR, release on failure
  src/setup/driver.rs            Driver: broker handles, fb VA, width, height, stride
  src/discover.rs                find_bga: PCI vendor/device/class + two-MMIO-BAR match
  src/dispi/set_mode.rs          the DISPI mode-set (disable, xres, yres, bpp, enable+lfb)
  src/dispi/dispi_off.rs         DISPI index -> MMIO byte offset (0x500 + index*2)
  src/dispi/clear.rs             solid-colour framebuffer fill
  src/regs.rs                    Regs: volatile 16-bit register accessors
  src/handles.rs                 BrokerHandles: unmap both BARs and release on drop
  src/error.rs                   BgaError, exit_code
  src/constants.rs               PCI identity, BAR indices, DISPI indices, mode, clear colour
  Cargo.toml                     crate and binary name, panic=abort
  README.md                      parked status and the promotion checklist
  userland/libc/src/broker/      the mk_device_*, mk_mmio_*, and mk_pci_config_write wrappers it calls
  userland/libc/src/syscall/numbers/broker.rs   the tag-encoded broker syscall numbers
  src/capabilities/types.rs      the capability bits a promoted manifest would declare
  userland/capsule_driver_virtio_gpu/Capsule.mk   the sibling driver's manifest, as the promotion model
  docs/subsystems/hardware-broker/   the claim, mmio, and pio grant contracts
```

Every reference above is verified against those trees.

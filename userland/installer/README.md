# The Installer Capsule

The installer is the keystone that turns a request into a running process. It takes either a capsule name
or a full artifact set, marshals the four blobs a capsule image needs, and hands them to the kernel's
verified-load syscall, which runs the entire trust chain before anything spawns. It is the capsule that
loads every other capsule, and it carries only the authority that job needs: it holds
no keys, verifies nothing itself, and defers every signature, manifest, and attestation check to the
kernel. Its mask adds `SpawnBroker`, so a capsule it loads on a requester's behalf is attributed to the
requester's pid rather than the installer's, and `FileSystem` for the store path; the verification itself
stays entirely in the kernel. Its job is to move bytes into `mk_capsule_load` and relay the kernel's verdict.

Its source is organized into two top-level modules, and this documentation mirrors that structure one page
per pillar so a page can be read beside the folder it describes.

## Identity

Everything the kernel and the service registry need to name and reach the installer comes from its
`Capsule.mk` and its kernel-side spawn record. The two are kept in lockstep: the kernel spawn constants
mirror the `Capsule.mk` fields exactly.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `installer` | `userland/capsule_installer/Capsule.mk:7` |
| Service handle | `installer` | `Capsule.mk:8`, `src/userspace/capsule_installer/spawn.rs:28` |
| Namespace | `systems.nonos.installer` | `Capsule.mk:14` |
| Service endpoint | `service:4112:installer` | `Capsule.mk:15`, `spawn.rs:29` |
| Reply endpoint | `reply:4113:endpoint.4294967313` | `Capsule.mk:16`, `spawn.rs:30`, `spawn.rs:31` |
| Capability mask | `0x800059` | `Capsule.mk:20` |
| Binary name | `installer` | `Capsule.mk:11` |
| Kernel mirror | `src/userspace/capsule_installer` | `Capsule.mk:19` |

The reply inbox name `endpoint.4294967322` is the decimal form of `0x1_0000_001A`, the constant
`KERNEL_REPLY_ENDPOINT` the server sends every reply to (`src/protocol/types.rs:30`). The value moved
from `0x1_0000_0011` after that collided with the NVMe driver's reply inbox; it must stay unique across
every capsule (`userland/capsule_installer/src/protocol/types.rs:26`). Requests arrive on inbox `0`, the
service port.

The mask `0x800059` decomposes into five bits, checked against `src/capabilities/types/defs.rs`:

| Bit | Value | Grants |
|-----|-------|--------|
| CoreExec | `0x000001` | run as a process |
| IPC | `0x000008` | send and receive on its endpoints (`mk_ipc_*`) |
| Memory | `0x000010` | map its own heap and stack |
| FileSystem | `0x000040` | the filesystem-capability gate |
| SpawnBroker | `0x800000` | attribute a loaded capsule to the requester's pid, not its own |

```
  0x800059 = 0x000001 + 0x000008 + 0x000010 + 0x000040 + 0x800000
```

`SpawnBroker` is the bit that lets the installer load a capsule on a requester's
behalf and attribute the child process to that requester's pid instead of the
installer's own, so the requester can `mk_wait`/`mk_kill`/`mk_proc_input`/
`mk_proc_output` the process it asked for
(`userland/capsule_installer/Capsule.mk:17`). The installer still holds no
`Crypto` bit, so it verifies nothing itself; the verification is entirely the
kernel's, on the `mk_capsule_load` path, which the kernel gates on the trust
chain rather than on the installer's caps. That separation is the whole basis of
the [verified-load](verified-load.md) argument: the installer moves bytes and
attributes the child, but the decision to run is the kernel's. It holds no
`Network`, `Driver`, `Mmio`, `Irq`, `Dma`, or `Pio`, so a bug in it cannot reach
hardware or the wire, and every payment settlement or name lookup it performs is
an IPC request to a capsule that does hold that right, checked at that boundary.

## The two pillars

The source under `userland/capsule_installer/src/` is two top-level modules, and the documentation is one
page each. A request comes in on the service port, the `protocol` codec splits it into a header and a
payload, the `server` dispatches one of four operations, and a load operation drives the request through
`mk_capsule_load` into the kernel's verified-spawn path.

```
  wire  ->  protocol/  ->  server/  ->  mk_capsule_load  ->  kernel verified spawn
  frame     the codec     dispatch      the syscall          re-verifies everything
            + ops         + handlers    (a request)          (the real gate)
```

| Page | Mirrors | What it covers |
|------|---------|----------------|
| [operations.md](operations.md) | `src/protocol/` and `src/server/` | The wire frame, the four operations (healthcheck, install, load-from-store, load-by-name), the dispatch table, the handlers, name validation, and the payment-admission call. |
| [verified-load.md](verified-load.md) | the load path from `mk_capsule_load` to `spawn_verified` | Why installing is safe: the syscall, the kernel-side handler, the manifest signature, ceiling, and grant checks, and why requested caps are bounded not granted. |
| [contributing.md](contributing.md) | the whole tree | Where to work, how to add an operation, the build and sign steps, and the code standards. |
| [debugging.md](debugging.md) | runtime | The boot marker, install-denied, capsule-not-found, and the other failure modes with where to look. |

## Lifecycle

The installer is spawned as part of the desktop-services fleet at boot. The plan calls `spawn_installer`
(gated on the `nonos-capsule-installer` feature), which runs `boot::capsule("INSTALLER", "installer", ...)`
and verifies the embedded ELF, cert, manifest, and attestation against the baked trust anchor before
registering `installer` on port 4112 (`src/userspace/init/spawn_plan/desktop_services.rs:21`, `:35`, `:37`,
`src/userspace/capsule_installer/spawn.rs:35`). A successful spawn prints `[INSTALLER] capsule spawned` on
the boot log; the [debugging](debugging.md) page covers what each later marker means.

`_start` initializes the heap and calls `server::run`, exiting with code 1 if the heap fails to initialize
(`src/main.rs:28`). Under the `nonos-autorun-install` feature the server first runs a headless
self-verification that loads the staged `std_proof` and `rg` packages through the same verified-load
syscall, so their output lands on serial without a user opening the GUI terminal first (`Capsule.mk:13`,
`Cargo.toml:26`, `src/server/selfinstall.rs:30`). It is a build-time feature of the capsule, not an
operation callers can invoke. The server then enters the request loop: receive one message on inbox `0`,
decode the header, dispatch one operation, reply to `KERNEL_REPLY_ENDPOINT` (`src/server/runner.rs:26`).

## Source map

```
  userland/capsule_installer/src/main.rs        _start -> heap_init -> server::run; the two modules
  userland/capsule_installer/src/protocol/      the wire codec and the op/errno constants
  userland/capsule_installer/src/server/        the loop, dispatch, discovery, and the four handlers
  userland/capsule_installer/Capsule.mk         slug, handle, ports, capability mask, kernel mirror
  src/capabilities/types/defs.rs                     the capability bits behind the mask
  src/userspace/capsule_installer/spawn.rs      the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/desktop_services.rs   the desktop-fleet spawn entry
```

Everything here is drawn from `userland/capsule_installer/` (the capsule source and its `Capsule.mk`),
`src/capabilities/types/defs.rs` (the capability bits), and the kernel spawn mirror under
`src/userspace/capsule_installer/`. Every reference above is verified against those trees.

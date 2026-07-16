# capsule_policy (full reference)

`capsule_policy` is the live settings store for NONOS: a typed, RAM-resident key-value store holding
every user preference, system toggle, and kernel security switch the desktop reads and the setup wizard
and settings app write. Reads are open to any capsule that can speak IPC; writes are gated to two named
apps. A small set of settings are mirrored into the kernel, and when one of those changes the capsule
pushes the new value across an admin syscall so the change takes effect rather than merely being
recorded. Service `policy` on port 4108 with a reply inbox on 4109, capability mask `0x219`. The source is
`userland/capsule_policy/` and the shared wire format is `userland/policy_proto/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [Field reference](#field-reference)
- [Operation reference](#operation-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The policy capsule is the single source of truth for runtime configuration. When the desktop shell needs
the brightness level, the theme, whether animations are on, or whether the machine is in anonymous mode,
it reads that field from `policy`. When the settings app or the first-boot setup wizard changes a
setting, it writes that field to `policy`. Nothing else on the system may write it.

The store lives entirely in RAM behind a spinlock (`src/store/state.rs:22`); it is not backed by a file
and does not persist across a reboot, so every boot starts from the compiled-in defaults
(`src/store/defaults/store.rs:21`). Reads and writes are typed: each field has a fixed kind (bool, u8,
i8, or string) and the wire refuses a request whose kind or length does not match
(`src/store/types.rs`, `userland/policy_proto/src/field_kind.rs:20`).

A subset of fields is not just user-facing state but is mirrored into the running kernel. For those, a
successful write is followed by an `AdminPolicyPush` syscall that carries the new value into the kernel
(`src/push/raw.rs:19`), and at startup the capsule seeds the kernel with the current values of exactly
those fields so the two agree from boot (`src/push/seed.rs:22`). This is the reason the capsule holds the
`Admin` capability, and it is the whole basis of the security discussion below.

## Identity

Everything the kernel and the service registry need to name and reach the capsule comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `policy` | `Capsule.mk:5` |
| Service handle | `policy` | `Capsule.mk:6`, `src/userspace/capsule_policy/spawn.rs:27` |
| Namespace | `systems.nonos.policy` | `Capsule.mk:11` |
| Service endpoint | `service:4108:policy` | `Capsule.mk:12`, `spawn.rs:28`, `spawn.rs:39` |
| Reply endpoint | `reply:4109:endpoint.policy.reply` | `Capsule.mk:13`, `spawn.rs:29`, `spawn.rs:30` |
| Capability mask | `0x219` | `Capsule.mk:15`, `spawn.rs:32` |
| Binary name | `policy` | `Capsule.mk:9` |
| Kernel mirror | `src/userspace/capsule_policy` | `Capsule.mk:16` |

The service name and both ports are also fixed in the shared proto so any client agrees with the server:
`POLICY_SERVICE_NAME = b"policy"`, `POLICY_SERVICE_PORT = 4108`, `POLICY_REPLY_PORT = 4109`
(`userland/policy_proto/src/service.rs:17`).

The mask `0x219` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x0001  CoreExec   bit()   1     types.rs:56
  0x0008  IPC        bit()   8     types.rs:59
  0x0010  Memory     bit()  16     types.rs:60
  0x0200  Admin      bit() 512     types.rs:65
  ------
  0x0219  = 1 + 8 + 16 + 512
```

The kernel spawn path requests exactly those four capabilities and no others (`requested_caps: 0x219`,
`src/userspace/capsule_policy/spawn.rs:32`, `spawn.rs:47`). There is no `FileSystem` bit (the store is a
RAM struct, not a file), no `Network`, no `Crypto`, and no hardware capability of any kind (no `Driver`,
`Mmio`, `Irq`, `Dma`, `Pio`). The load-bearing bit is `Admin`, which is what gates the `AdminPolicyPush`
syscall that mirrors fields into the kernel (`src/syscall/contract/cap_table/admin.rs:24`,
`src/capabilities/token/types.rs:121`).

## Field reference

A field is a `u32` discriminant grouped by high byte: `0x01xx` user preferences, `0x02xx` kernel
security, `0x03xx` system identity (`userland/policy_proto/src/field.rs:17`,
`userland/policy_proto/src/category.rs:27`). `decode_field` maps a discriminant to a `Field` and rejects
any unknown value with `E_INVAL`, so only defined fields are addressable
(`userland/policy_proto/src/field_decode.rs:19`). Each field has a fixed kind
(`userland/policy_proto/src/field_kind.rs:20`), a store slot (`src/store/types.rs:26`), and a compiled-in
default (`src/store/defaults/store.rs:21`). The defaults below are the values every boot starts from.

User preferences (`0x01xx`):

| Field | Id | Kind | Default | Source (default) |
|---|---|---|---|---|
| Brightness | 0x0101 | u8 (max 100) | 80 | `defaults/store.rs:23`, max `field_max.rs:25` |
| MouseSensitivity | 0x0102 | u8 (max 4) | 2 | `defaults/store.rs:24`, max `field_max.rs:26` |
| SoundEnabled | 0x0103 | bool | true | `defaults/store.rs:25` |
| AnonymousMode | 0x0104 | bool | true | `defaults/store.rs:26` |
| NymEnabled | 0x0105 | bool | false | `defaults/store.rs:27` |
| Theme | 0x0106 | u8 (enum) | 0 | `defaults/store.rs:28` |
| KeyboardLayout | 0x0107 | u8 (enum) | 0 | `defaults/store.rs:29` |
| AutoWipe | 0x0108 | bool | true | `defaults/store.rs:30` |
| Timezone | 0x0109 | i8 (-12..=14) | 0 | `defaults/store.rs:31`, range `store/set_i8.rs:25` |
| ScreenTimeout | 0x010A | u8 (max 240) | 0 | `defaults/store.rs:32`, max `field_max.rs:27` |
| Language | 0x010B | u8 (enum) | 0 | `defaults/store.rs:33` |
| DeveloperMode | 0x010C | bool | false | `defaults/store.rs:34` |
| HardwareCrypto | 0x010D | bool | true | `defaults/store.rs:35` |
| ZkAttestation | 0x010E | bool | true | `defaults/store.rs:36` |
| SystemKeysGenerated | 0x010F | bool | false | `defaults/store.rs:37` |
| NotificationsEnabled | 0x0110 | bool | true | `defaults/store.rs:38` |
| HighContrast | 0x0111 | bool | false | `defaults/store.rs:39` |
| FontSize | 0x0112 | u8 (enum) | 1 | `defaults/store.rs:40` |
| AutoLockTimeout | 0x0113 | u8 (max 240) | 5 | `defaults/store.rs:41`, max `field_max.rs:28` |
| WifiAutoconnect | 0x0114 | bool | true | `defaults/store.rs:42` |
| AnimationsEnabled | 0x0115 | bool | true | `defaults/store.rs:43` |
| CursorSize | 0x0116 | u8 (enum) | 1 | `defaults/store.rs:44` |
| Wallpaper | 0x0117 | u8 (enum) | 52 | `defaults/store.rs:45` |
| ClockFormat24 | 0x0118 | bool | true | `defaults/store.rs:46` |

Kernel security (`0x02xx`), all bool:

| Field | Id | Default | Source (default) |
|---|---|---|---|
| KernelAslr | 0x0201 | true | `defaults/store.rs:47` |
| KernelStackGuard | 0x0202 | true | `defaults/store.rs:48` |
| KernelNxBit | 0x0203 | true | `defaults/store.rs:49` |
| KernelSmep | 0x0204 | true | `defaults/store.rs:50` |
| KernelSmap | 0x0205 | true | `defaults/store.rs:51` |
| KernelDebug | 0x0206 | false | `defaults/store.rs:52` |
| KernelSerial | 0x0207 | true | `defaults/store.rs:53` |
| KernelWatchdog | 0x0208 | false | `defaults/store.rs:54` |
| KernelPreempt | 0x0209 | true | `defaults/store.rs:55` |
| KernelHugepages | 0x020A | false | `defaults/store.rs:56` |
| KernelIommu | 0x020B | true | `defaults/store.rs:57` |
| KernelSeccomp | 0x020C | true | `defaults/store.rs:58` |

System identity (`0x03xx`), string (`StringField`: `[u8; 64]` plus a `len`, `src/store/types.rs:19`):

| Field | Id | Kind | Default | Source (default) |
|---|---|---|---|---|
| Hostname | 0x0301 | str | `nonos` | `defaults/store.rs:59`, `defaults/constants.rs:17` |
| DomainName | 0x0302 | str | empty | `defaults/store.rs:60`, `defaults/empty_string.rs:19` |

That is 24 user + 12 kernel + 2 identity = 38 fields, one per discriminant in the enum
(`userland/policy_proto/src/field.rs:19`) and one per slot in the store struct
(`src/store/types.rs:26`). The u8 enum fields (Theme, Wallpaper, KeyboardLayout, Language, FontSize,
CursorSize) are validated against a label table whose length bounds the value; the numeric u8 fields
(Brightness, MouseSensitivity, ScreenTimeout, AutoLockTimeout) are bounded by an explicit max, and any
u8 with no declared bound accepts the full range (`userland/policy_proto/src/field_max.rs:20`,
`userland/policy_proto/src/enum_table.rs:25`).

## Operation reference

The protocol has exactly two operations. Both name a `Field` and a `kind` in a fixed 12-byte header and
carry a per-kind payload (`userland/policy_proto/src/hdr.rs:17`).

| Op | Opcode | Direction | Payload | Source |
|---|---|---|---|---|
| `OP_GET` | 0x0001 | request | none; reply carries the value | `ops.rs:17`, `server/handle_get.rs:21` |
| `OP_SET` | 0x0002 | request | the new value, per kind | `ops.rs:18`, `server/handle_set.rs:40` |

There is no separate push or subscribe opcode on this service. The "push" side is outbound: it is the
capsule calling the kernel's `AdminPolicyPush` syscall after a mirrored field changes, described under
[protocol and IPC](#protocol-and-ipc) and [architecture](#architecture-and-lifecycle). Readers do not
subscribe; they poll a `get` when they need a value.

### get

`handle_get::dispatch` routes by the field's kind to the matching handler
(`src/server/handle_get.rs:21`). Each handler reads the store and replies with the value, or `E_NOT_FOUND`
if the field is not one this kind's getter knows (`src/server/handlers/get_bool.rs:22`):

- bool: one payload byte, `1` or `0` (`server/handlers/get_bool.rs:24`, store `store/get_bool.rs:21`).
- u8: one payload byte (`server/handlers/get_u8.rs:22`, store `store/get_u8.rs:21`).
- i8: one payload byte, the value reinterpreted (`server/handlers/get_i8.rs:22`, store
  `store/get_i8.rs:21`).
- str: up to `STRING_CAP = 64` payload bytes with no trailing NUL (`server/handlers/get_str.rs:22`, store
  `store/get_str.rs:22`).

### set

`handle_set::dispatch` first applies the write gate (below), then routes by kind to the matching setter
(`src/server/handle_set.rs:40`). Each setter length-checks the payload, mutates the store, mirrors the
field if it is a kernel-mirrored one, and replies:

- bool: payload must be exactly one byte or `E_BAD_LEN`; nonzero is true; on success mirrors through
  `push::on_bool_set` and replies empty; an unknown bool field is `E_INVAL`
  (`src/server/handlers/set_bool.rs:23`).
- u8: exactly one byte or `E_BAD_LEN`; the value is range-checked in the store (`E_INVAL` if over the
  field max) before the mutation (`src/server/handlers/set_u8.rs:22`, `src/store/set_u8.rs:22`).
- i8: exactly one byte or `E_BAD_LEN`; the store enforces `-12..=14` for Timezone (`E_INVAL` otherwise);
  on success mirrors through `push::on_i8_set` (`src/server/handlers/set_i8.rs:23`,
  `src/store/set_i8.rs:25`).
- str: at most `STR_MAX = 63` bytes or `E_BAD_LEN`; the store further caps at `STRING_CAP = 64` and
  validates the byte set to `[A-Za-z0-9._-]` (`E_INVAL` otherwise); on success mirrors through
  `push::on_string_set` (`src/server/handlers/set_str.rs:23`, `src/store/set_str.rs:23`,
  `src/store/str_validate.rs:17`).

### errors

Every failure is a header-only reply carrying the op, field, kind, and a status word
(`src/server/respond/err.rs:21`).

| Status | Value | When | Source |
|---|---|---|---|
| `E_OK` | 0 | success reply | `errno.rs:17`, `respond/ok.rs:24` |
| `E_INVAL` | 22 | unknown op, unknown field discriminant, body longer than the frame, or a value the store rejects | `errno.rs:18`, `server/runner.rs:42`, `server/runner.rs:48`, `server/runner.rs:55` |
| `E_BAD_LEN` | 90 | payload length wrong for the kind, or a reply payload that would exceed the buffer | `errno.rs:19`, `server/handlers/set_bool.rs:25`, `respond/ok.rs:26` |
| `E_NOT_FOUND` | 91 | `get` for a field this kind's getter does not carry | `errno.rs:20`, `server/handlers/get_bool.rs:28` |
| `E_ACCES` | 93 | `set` from any caller that is not a trusted setter | `errno.rs:22`, `server/handle_set.rs:42` |

`E_WRONG_KIND` (92) is defined in the proto but is not currently raised by a handler; a kind mismatch
surfaces as `E_BAD_LEN` or `E_INVAL` through the per-kind length and field checks
(`userland/policy_proto/src/errno.rs:21`).

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` initializes the heap, registers the service, seeds the kernel
with the mirrored defaults, and enters the request loop (`src/main.rs:30`). The four top-level modules are
`bootstrap` (service registration), `store` (the field store), `server` (the request loop and handlers),
and `push` (the kernel mirror) (`src/main.rs:22`).

The store is a single `Store` struct with one slot per field, held behind a global `spin::Mutex`
initialized from the const defaults (`src/store/types.rs:26`, `src/store/state.rs:22`,
`src/store/defaults/store.rs:21`). Access is split one unit per file: `get_bool`, `get_u8`, `get_i8`,
`get_str` and their `set_*` counterparts, each matching a `Field` to its slot and returning `None`/`false`
for a field outside its kind (`src/store/get_bool.rs`, `src/store/set_u8.rs`, and siblings).

The server loop polls the endpoint, decodes the 12-byte header, bounds-checks the body against the frame,
decodes the field, and dispatches on the op (`src/server/runner.rs:23`):

```
  run(endpoint):
      loop:
          n = recv::poll(endpoint, buf, &sender)      // mk_ipc_recv_from, non-blocking
          if n <= 0: yield; continue
          if n < HDR_LEN: continue                     // 12-byte header required
          hdr = Header::decode(buf)                    // op, field, kind, status, payload_len
          if HDR_LEN + hdr.payload_len > n: E_INVAL    // body must fit the frame
          field = decode_field(hdr.field)              // u32 -> Field; else E_INVAL
          match hdr.op:
              OP_GET -> handle_get::dispatch(sender, field)
              OP_SET -> handle_set::dispatch(sender, field, body)
              _      -> E_INVAL
```

The reply goes straight back to the sender the recv reported, through `mk_ipc_reply`
(`src/server/reply.rs:19`).

The kernel mirror is the `push` module. Only four fields are mirrored, not the whole store:
`KernelPreempt` (bool), `Timezone` (i8), `Hostname` and `DomainName` (str). `on_bool_set` forwards only
`KernelPreempt` (`src/push/on_bool_set.rs:22`); `on_i8_set` forwards only `Timezone`
(`src/push/on_i8_set.rs:22`); `on_string_set` forwards `Hostname` and `DomainName`
(`src/push/on_string_set.rs:22`). Each maps the field to a small kernel-side id and kind and calls
`raw::submit`, which is the `mk_admin_policy_push` syscall (`src/push/kernel_field.rs:17`,
`src/push/raw.rs:19`). The kernel end validates the id and kind and applies the value
(`src/syscall/dispatch/router/admin/policy_push/entry.rs:25`,
`src/sys/policy/field_id.rs:19`). The other kernel-security toggles (ASLR, SMEP, SMAP, NX, IOMMU,
watchdog, seccomp, and the rest) are recorded in the store and served to readers, but the current build
does not push them into the kernel; only `KernelPreempt` among that group is mirrored.

Lifecycle:

1. `heap_init` sets up the allocator; failure exits with code 1 (`src/main.rs:31`).
2. `bootstrap::register` calls `mk_service_register("policy", 4108)`; failure exits with code 2
   (`src/main.rs:34`, `src/bootstrap/register.rs:21`, `src/bootstrap/port.rs:17`).
3. `push::seed_kernel` reads the current `KernelPreempt`, `Timezone`, `Hostname`, and `DomainName` and
   pushes each into the kernel so the two agree from boot (`src/main.rs:37`, `src/push/seed.rs:22`).
4. `server::run` enters the poll loop and never returns (`src/main.rs:38`, `src/server/runner.rs:23`).

The kernel spawns the capsule at boot behind the `nonos-capsule-policy` feature, verifying the embedded
ELF, id cert, manifest, and attestation against the baked trust anchor before it runs
(`src/userspace/init/spawn_plan/core.rs:63`, `src/userspace/capsule_policy/spawn.rs:34`).

## Protocol and IPC

Inbound, the capsule serves the two-op request protocol on `service:4108:policy` and replies on
`reply:4109:endpoint.policy.reply`. The header is 12 bytes, little-endian: `op(2) | field(4) | kind(1) |
pad(1) | status(2) | payload_len(2)` (`userland/policy_proto/src/hdr.rs:17`). The kinds are `KIND_BOOL 1`,
`KIND_U8 2`, `KIND_I8 3`, `KIND_STR 4` (`userland/policy_proto/src/kind.rs:17`). The whole frame is
bounded by `IPC_PAYLOAD_MAX = 512` and a reply payload by `IPC_PAYLOAD_MAX - HDR_LEN`
(`userland/policy_proto/src/limits.rs:18`, `src/server/respond/ok.rs:22`).

Outbound, the capsule makes one call, and only to the kernel: `mk_admin_policy_push(field_id, kind,
ptr, len)`, the `AdminPolicyPush` syscall (`APPS`), which the kernel admits only for a caller that holds
`Admin` (`src/push/raw.rs:19`, `src/syscall/numbers/defs.rs:42`,
`src/syscall/contract/cap_table/admin.rs:24`). The kernel-side ids are its own small enum
(`KernelPreempt 0x0001`, `TimezoneOffset 0x0002`, `Hostname 0x0003`, `DomainName 0x0004`), distinct from
the wire `Field` discriminants (`src/sys/policy/field_id.rs:19`).

### The write gate

This is the property that makes the store trustworthy. `handle_set::dispatch` resolves the two allowed
setter names through the kernel service registry and compares each resolved pid to the sender before it
touches the store (`src/server/handle_set.rs:40`):

```
  SETTERS = [ b"app.settings", b"app.setup_wizard" ]          handle_set.rs:23

  is_trusted_setter(sender):
      any name in SETTERS where mk_service_lookup(name) -> (port, pid)
      with pid != 0 and pid == sender                          handle_set.rs:25, :36

  dispatch(pid, field, payload):
      if not is_trusted_setter(pid):
          respond E_ACCES                                       handle_set.rs:41
          return
      route by kind_of(field) -> set_bool / set_u8 / set_i8 / set_str
```

Only the settings app and the setup wizard, named `app.settings` and `app.setup_wizard` and resolved to
their live pids through `mk_service_lookup`, may call `set` (`src/server/handle_set.rs:23`,
`src/server/handle_set.rs:37`). Every other caller's `set` returns `E_ACCES` before the store is read or
written (`src/server/handle_set.rs:42`). `get` has no such gate: any caller that can reach the endpoint
may read any field. See [settings.md](settings.md) and [setup-wizard.md](setup-wizard.md) for the two
writers.

## Security analysis

The write gate is the whole point. Any capsule can read a setting, but a random capsule cannot flip one,
so a compromised reader can observe configuration but cannot change the theme, the hostname, or a kernel
toggle. The gate resolves the two writer names to pids at the moment of the call, so a caller cannot spoof
the identity by claiming a name; it has to actually be the pid the registry returns for `app.settings` or
`app.setup_wizard` (`src/server/handle_set.rs:37`).

What each side of a compromise can do:

- A compromised reader (any capsule holding IPC) can `get` every field, including the kernel-security
  toggles and the identity strings. It cannot `set` anything; the gate returns `E_ACCES` before any
  mutation (`src/server/handle_set.rs:41`).
- A compromised writer (`app.settings` or `app.setup_wizard`) can `set` any field, including
  `KernelPreempt`, whose change is pushed into the running kernel. This is a real trust concentration: the
  gate is coarse, keyed on the service name rather than a per-field capability, so either writer can write
  the whole policy surface. It is stated here rather than hidden.

The typing and bounds limit the blast radius even for a writer. Values are range-checked
(`src/store/set_u8.rs:22`), the timezone is bounded (`src/store/set_i8.rs:25`), and strings are capped at
64 bytes and restricted to `[A-Za-z0-9._-]`, so a hostname cannot smuggle arbitrary bytes into whatever
reads it (`src/store/str_validate.rs:17`).

The capability mask is the second boundary. The mask is `0x219` = CoreExec, IPC, Memory, Admin
(`Capsule.mk:15`, `src/userspace/capsule_policy/spawn.rs:32`). `Admin` is the only elevated bit, and it
buys exactly one thing: the right to call `AdminPolicyPush` (`src/syscall/contract/cap_table/admin.rs:24`).
The mask holds no FileSystem, no Crypto, no Network, and no hardware capability (`Driver`, `Mmio`, `Irq`,
`Dma`, `Pio`), so the capsule that can flip a kernel security bit cannot itself touch the hardware that
bit protects, cannot read a block device, and cannot open a socket. Isolation from every other capsule is
the kernel's: `policy` is a CPL 3 user binary verified and enrolled at spawn like any other capsule
(`src/userspace/capsule_policy/spawn.rs:34`), and its only privileged edge is the single, kind-and-id
validated admin syscall.

Honest gaps. The write gate is coarse (service name, not a capability), so a compromise of either writer
is a compromise of the whole store. The store is the capsule's own copy in RAM, so if the kernel changed
a toggle by another path the copy could go stale, and only the four mirrored fields flow the other way.
There is no audit log of policy changes and no transactional multi-field update; each `set` is
independent. And of the twelve kernel-security fields, only `KernelPreempt` is actually pushed into the
kernel today; the rest are recorded and readable but not yet wired to a kernel effect
(`src/push/on_bool_set.rs:22`).

## How to contribute

The source lives at `userland/capsule_policy/`, with the shared wire format at `userland/policy_proto/`.
The store is under `src/store/`, the request loop and handlers under `src/server/`, the kernel mirror
under `src/push/`, and service registration under `src/bootstrap/`.

To add a new policy field:

1. Add the discriminant to the `Field` enum, keeping it in its category range (`0x01xx` user, `0x02xx`
   kernel, `0x03xx` identity) (`userland/policy_proto/src/field.rs:19`), and add the matching arm to
   `decode_field` (`userland/policy_proto/src/field_decode.rs:19`).
2. Declare its kind in `kind_of` (`userland/policy_proto/src/field_kind.rs:20`), and if it is a bounded
   u8, add its max to `max_of` or its label table to `enum_table`
   (`userland/policy_proto/src/field_max.rs:20`, `userland/policy_proto/src/enum_table.rs:25`).
3. Add the slot to the `Store` struct (`src/store/types.rs:26`) and its default to the const `store()`
   (`src/store/defaults/store.rs:21`).
4. Wire it into the matching store getter and setter arm (for a bool, `src/store/get_bool.rs:21` and
   `src/store/set_bool.rs:21`).
5. If the field must reach the kernel, add a kernel-side id in `kernel_field.rs`, an arm to the matching
   `push::on_*_set`, and a seed line in `push::seed_kernel`, then handle the id on the kernel side in
   `src/sys/policy/field_id.rs` and the `policy_push` router (`src/push/kernel_field.rs:17`,
   `src/push/on_bool_set.rs:22`, `src/push/seed.rs:22`).

To build and sign the capsule, use the generated per-slug make targets, which the slug `policy` in
`Capsule.mk:5` expands from the shared template (`nonos-mk/capsule.mk:158`, included through
`userland/capsule_policy/Capsule.mk:18` and `Makefile:647`):

```
  make nonos-mk-policy              build the capsule ELF
  make nonos-mk-policy-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-policy-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-policy-keys   check the per-capsule signing keys exist
```

(`nonos-mk/capsule.mk:182`, `:261`, `:263`, `:184`.) There is no `policy`-specific desktop-image target;
the capsule is pulled into the desktop profiles through the shared verified-capsule lists in the Makefile
(`Makefile:723`, `Makefile:1076`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns an error status through `respond::err`, never a panic);
modular files, one unit per file, with `mod.rs` used only for re-exports (`src/store/mod.rs`,
`src/server/mod.rs`); and the AGPL header at the top of every source file, matching the header on every
existing module.

## Debugging

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints `[POLICY]
capsule spawned` (tag `POLICY`, message `capsule spawned`) from the boot log
(`src/userspace/init/spawn_plan/core.rs:67`, `src/userspace/init/capsule_boot/run.rs:29`,
`src/sys/boot_log/output.rs:33`). An absent line means the capsule never started, usually a signature,
manifest, or capability failure; the error path prints an `[ERROR]` line instead
(`src/userspace/init/capsule_boot/run.rs:32`).

One nuance specific to this capsule: at startup its `main.rs` runs `push::seed_kernel` before entering
the loop (`src/main.rs:37`), so a `[POLICY]` line means both that the service registered and that the
kernel was primed with the mirrored defaults. If `mk_service_register` failed the process would have
exited with code 2 before printing anything (`src/main.rs:34`).

Failure modes and where to look:

- A setting does not take effect. Reads and writes of a plain field only touch the RAM store, so a change
  that "does not take effect" is usually a reader that has not re-read the field, not a capsule fault.
  For the four mirrored fields (`KernelPreempt`, `Timezone`, `Hostname`, `DomainName`), the write also
  fires an `AdminPolicyPush`; a store update that lands but a kernel effect that does not is a
  push-side symptom, so check the admin syscall and its kernel handler
  (`src/push/on_bool_set.rs:22`, `src/syscall/dispatch/router/admin/policy_push/entry.rs:25`). Any other
  kernel-security toggle is recorded but not pushed by the current build, so expecting a kernel effect
  from, for example, `KernelSmep` is expecting behavior that is not wired yet.
- A write is denied. A `set` from anything but `app.settings` or `app.setup_wizard` returns `E_ACCES`;
  that is the write gate doing its job (`src/server/handle_set.rs:42`). If the legitimate writer is being
  denied, the suspect is service resolution: `mk_service_lookup` must return that writer's live pid, so
  confirm the settings app or wizard actually registered its service name (`src/server/handle_set.rs:37`).
- A request is rejected. `E_INVAL` means an unknown op, an unknown field discriminant, a body that runs
  past the frame, or a value the store refuses (over a max, out of the timezone range, or a bad string
  byte) (`src/server/runner.rs:42`, `:48`, `:55`). `E_BAD_LEN` means the payload length did not match the
  kind (a bool or numeric that was not exactly one byte, or a string over 63 bytes)
  (`src/server/handlers/set_bool.rs:25`, `src/server/handlers/set_str.rs:25`). `E_NOT_FOUND` means a
  `get` reached a getter that does not carry that field for its kind (`src/server/handlers/get_bool.rs:28`).
- Malformed input is dropped. A frame shorter than the 12-byte header is silently skipped rather than
  answered, and a header that fails to decode is skipped; only a decodable header with a bad op or field
  gets an error reply (`src/server/runner.rs:32`, `:36`).

## Source map

```
  userland/capsule_policy/src/main.rs                 _start: heap, register, seed, run
  userland/capsule_policy/src/bootstrap/              service name, port, mk_service_register
  userland/capsule_policy/src/server/runner.rs        the poll/decode/dispatch loop
  userland/capsule_policy/src/server/handle_get.rs    typed get dispatch
  userland/capsule_policy/src/server/handle_set.rs    the trusted-setter gate + typed set dispatch
  userland/capsule_policy/src/server/handlers/        per-kind get/set handlers (length checks, replies)
  userland/capsule_policy/src/server/respond/         ok / err reply encoders
  userland/capsule_policy/src/store/types.rs          the Store struct: 38 slots + StringField
  userland/capsule_policy/src/store/state.rs          the global STORE mutex
  userland/capsule_policy/src/store/defaults/         compiled-in defaults, hostname, empty string
  userland/capsule_policy/src/store/{get,set}_*.rs    per-kind field access + range checks
  userland/capsule_policy/src/store/str_validate.rs   the [A-Za-z0-9._-] byte check
  userland/capsule_policy/src/push/                   the kernel mirror (on_bool/i8/string_set, seed, raw)
  userland/capsule_policy/Capsule.mk                  slug, handle, ports, capability mask, kernel mirror
  userland/policy_proto/src/field.rs                  the Field enum (38 discriminants)
  userland/policy_proto/src/field_decode.rs           u32 -> Field (E_INVAL on unknown)
  userland/policy_proto/src/field_kind.rs             Field -> kind
  userland/policy_proto/src/{field_max,enum_table}.rs u8 bounds and enum label tables
  userland/policy_proto/src/{ops,kind,errno,hdr,limits,service}.rs   the wire constants and header
  src/userspace/capsule_policy/spawn.rs               the kernel-side verified spawn
  src/userspace/init/spawn_plan/core.rs               the boot-fleet spawn entry (POLICY)
  src/sys/policy/field_id.rs                          the kernel-side mirrored field ids
  src/syscall/dispatch/router/admin/policy_push/      the AdminPolicyPush handler (Admin-gated)
```

Every reference above is verified against those trees.

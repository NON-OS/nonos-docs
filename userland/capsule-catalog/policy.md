# capsule_policy

`capsule_policy` is the system configuration store: a typed key-value store of every user- and
kernel-facing setting, from display brightness to the kernel's own security toggles. Reads are open to
any caller; writes are restricted to two trusted apps. When a kernel-relevant setting changes, the
capsule pushes the new value to the kernel so it takes effect. Service `policy` on port 4108, capability
mask `0x219`. The source is `userland/capsule_policy/`.

## Contents

- [The server loop](#the-server-loop)
- [The field protocol](#the-field-protocol)
- [The store](#the-store)
- [Typed get and set](#typed-get-and-set)
- [The write gate](#the-write-gate)
- [Kernel propagation](#kernel-propagation)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop

`main.rs:28` initializes the heap, registers the service, seeds the kernel with the default policy
(`push::seed_kernel`), and runs the loop (`src/server/runner.rs:23`) over a 64 KiB stack buffer:

```
  run(endpoint):
      loop:
          n = recv::poll(endpoint, buf, &sender)
          hdr = Header::decode(buf)                    // op, field, kind, payload_len
          field = decode_field(hdr.field)              // u32 -> Field enum; else E_INVAL
          match hdr.op:
              OP_GET -> handle_get::dispatch(sender, field)
              OP_SET -> handle_set::dispatch(sender, field, payload)
              _      -> E_INVAL
```

The protocol is the shared `nonos_policy_proto`: a request names a `Field` and a `kind`, and the reply
goes directly to the sender.

## The field protocol

A request carries a `Field` (an enum discriminant) and a `kind`, one of `KIND_BOOL`, `KIND_U8`,
`KIND_I8`, or `KIND_STR`. The kind determines the payload: a `BOOL` is one byte, a `U8`/`I8` is one byte,
and a `STR` is up to `STRING_CAP = 64` bytes. `decode_field` rejects an unknown field discriminant with
`E_INVAL`, so only defined fields are addressable.

## The store

The `Store` (`src/store/types.rs:26`) is a single struct with a field per setting, held behind a global
mutex (`STORE`), grouped by concern:

```
  display / input   brightness, mouse_sensitivity, theme, keyboard_layout, font_size, cursor_size,
                    wallpaper, high_contrast, animations_enabled, clock_format24, language
  privacy           anonymous_mode, nym_enabled, auto_wipe, developer_mode
  system            sound_enabled, notifications_enabled, screen_timeout, auto_lock_timeout,
                    wifi_autoconnect, timezone, zk_attestation, hardware_crypto, system_keys_generated
  kernel security   kernel_aslr, kernel_stack_guard, kernel_nx_bit, kernel_smep, kernel_smap,
                    kernel_debug, kernel_serial, kernel_watchdog, kernel_preempt, kernel_hugepages,
                    kernel_iommu, kernel_seccomp
  identity          hostname, domainname   (StringField: [u8; 64] + len)
```

The kernel-security group is notable: policy is where the machine's ASLR, SMEP/SMAP, NX, IOMMU, seccomp,
watchdog, preempt, and debug/serial toggles are recorded, and a change to one is propagated to the kernel
(below).

## Typed get and set

`handle_get::dispatch` (`src/server/handle_get.rs:21`) routes by kind to `get_bool`, `get_u8`, `get_i8`,
or `get_str`, each reading the corresponding field from the store and replying with the value or
`E_NOT_FOUND`. `handle_set` mirrors it for writes, with a per-kind length check (a `BOOL` payload must be
exactly one byte) and a store mutation. `set_bool::set` (`src/store/set_bool.rs:21`) locks the store and
matches the `Field` to its boolean field; an unknown field for the kind returns false and the handler
replies `E_INVAL`.

## The write gate

Reads are open; writes are gated. `handle_set::dispatch` (`src/server/handle_set.rs:40`) checks the caller
against a trusted-setter list before mutating:

```
  is_trusted_setter(sender):
      SETTERS = [ "app.settings", "app.setup_wizard" ]
      any name in SETTERS resolves (via mk_service_lookup) to sender

  dispatch(pid, field, payload):
      if not is_trusted_setter(pid):  return E_ACCES
      match kind_of(field): set_bool / set_u8 / set_i8 / set_str
```

Only the [settings app](apps-and-proofs.md) and the [setup wizard](setup-wizard.md), identified by
resolving their service names to pids through the kernel service registry, may write policy. Any other
caller's set is `E_ACCES`.

## Kernel propagation

When a kernel-relevant field changes, the capsule pushes it to the kernel. `set_bool::handle` calls
`push::on_bool_set` (`src/push/on_bool_set.rs`) after a successful mutation, which forwards the new value
to the kernel for fields like `kernel_preempt`, so a policy change is not merely recorded but takes
effect. `push::seed_kernel`, called at startup, primes the kernel with the default policy so the two are
consistent from boot.

## Security analysis

- **Read/write asymmetry**: anyone can read a setting; only two named apps can write, so a random capsule
  cannot flip a kernel security toggle.
- **Typed and bounded**: fields are typed and length-checked, strings capped at 64 bytes.
- **Kernel coherence**: kernel-relevant changes propagate, so the store and the kernel do not silently
  diverge on a change made through policy.

The honest caveat is that the write gate is **coarse**: it trusts `app.settings` and `app.setup_wizard`
by service name and pid, not by a fine-grained capability, so a compromise of either app could write any
policy field, including a kernel security toggle. This is a real trust concentration, stated rather than
hidden.

The capability mask is `0x219` (`Capsule.mk`, `CAPSULE_REQUIRED_CAPS`), which decodes to CoreExec (1), IPC
(8), Memory (16), and Admin (512). The Admin bit is the load-bearing one and the reason this capsule is not
like the other config stores: it pushes kernel-relevant fields (`kernel_preempt`, the SMEP/SMAP/NX/IOMMU
toggles) into the kernel through `push::on_bool_set`, and that privileged propagation is what Admin grants.
The least-privilege reading is that Admin is the *only* elevated capability here: the mask holds no
FileSystem (policy is a RAM struct behind a mutex, not a file), no Crypto, no Network, and no hardware
capability at all (no Driver, Mmio, Irq, Dma, Pio), so the capsule that can flip a kernel security bit
cannot itself touch the hardware that bit protects. Its isolation from every other capsule is that the
write path is gated to two named apps resolved through the service registry, so a random capsule holding
IPC can *read* a setting but cannot *set* one. The honest boundary, stated in the security caveat above and
the [gaps](#honest-gaps), is that the write gate is by service name rather than a capability, so it is a
coarse trust concentration on those two apps.

## Debugging

The service is `policy` on port 4108 (`Capsule.mk`, `service:4108:policy`), and the capsule is spawned at
`spawn_plan/core.rs:67` (behind the `nonos-capsule-policy` feature) as `boot::capsule("POLICY", "policy",
...)` from `src/userspace/capsule_policy/`. It prints `[POLICY] capsule spawned` through
`capsule_boot::boot` on success, or a `[ERROR]` line with the `SpawnError` (framebuffer under
`NONOS_FBCONSOLE=1`). One nuance specific to this capsule: at startup its `main.rs` runs
`push::seed_kernel` before entering the loop, so a `[POLICY]` marker means both that the service registered
and that the kernel was primed with the default policy, and a boot where kernel security toggles read as
their defaults but a setting change never takes effect is a seed-ran-but-propagation-failed symptom rather
than a spawn failure. Once up, `mk_service_lookup("policy")` resolves for readers. The request-time failure
signatures are `E_INVAL` for an unknown field discriminant, an unknown op, or a kind/length mismatch (a
`BOOL` payload that is not exactly one byte), `E_NOT_FOUND` for a get on an unset field, and `E_ACCES` for
a set from any caller that is not `app.settings` or `app.setup_wizard`, the last being the write gate doing
its job.

## Honest gaps

Stated plainly: the write gate is by service name, not a capability mask (above); the store is the
capsule's copy, so if the kernel changes a toggle directly the copy can go stale; there is no audit log
of policy changes; an unknown op is a silent drop in some paths rather than an error reply; and there is
no transactional multi-field update, each set is independent.

## Source map

```
  userland/capsule_policy/src/server/runner.rs      the poll/decode/dispatch loop
  userland/capsule_policy/src/server/handle_get.rs   typed get dispatch
  userland/capsule_policy/src/server/handle_set.rs   the trusted-setter gate + typed set
  userland/capsule_policy/src/store/types.rs         the field store (39 fields + hostname/domainname)
  userland/capsule_policy/src/store/set_*.rs, get_*.rs   per-kind field access
  userland/capsule_policy/src/push/on_bool_set.rs, seed_kernel.rs   kernel propagation
```

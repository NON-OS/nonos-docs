# Capabilities and Tokens

A verified capsule is admitted to run, but admission says nothing about what it
may do once running. That is the job of capabilities. Every privileged action is
guarded by a capability bit, and a capsule holds a token that proves which bits
it owns. This page covers the bits, how they are declared and enforced, and the
token that carries them.

Read [capsules and the trust anchor](capsules-and-trust.md) first; the capability
set a capsule receives is the output of the verified-spawn pipeline described
there.

---

## The capability bits

A capability is a single bit in a `u64`. There are 22, defined as an enum whose
discriminants are the bit values themselves (`src/capabilities/types.rs`):

```
  CoreExec               1          IPC                    8
  IO                     2          Memory                 16
  Network                4          Crypto                 32
  FileSystem             64         Hardware               128
  Debug                  256        Admin                  512
  RegisterService        1024       GraphicsDisplayQuery   2048
  GraphicsSurfaceCreate  4096       GraphicsSurfaceMap     8192
  GraphicsPresent        16384      DeviceEnum             32768
  Driver                 65536      Mmio                   131072
  Irq                    262144     Dma                    524288
  Pio                    1048576    InputSource            2097152
```

The bits are grouped by the kind of authority they grant:

```
  baseline      CoreExec, IPC, Memory          run, message, allocate
  services      RegisterService, Crypto, IO    offer a service, use kernel crypto
  graphics      GraphicsDisplayQuery, SurfaceCreate, SurfaceMap, Present
  device        DeviceEnum, Driver             see and claim devices
  hardware      Mmio, Irq, Dma, Pio            the four broker grants
  input         InputSource                    post into the input ring
  privileged    Admin, Debug, Hardware         the dangerous bits, rarely granted
```

A driver capsule is the clearest example of a focused grant. The PS/2 input
driver requests exactly what it needs and nothing more
(`src/hardware/ps2_kbd_capsule/spawn.rs`):

```
  CoreExec | IPC | Memory | Driver | DeviceEnum | Pio | Irq | InputSource
```

It can run, message, allocate, claim its device, program ports and its IRQ line,
and post input. It cannot touch the network, the framebuffer, or another
device. If it tried, the syscall would return `EPERM`.

## Declaration and the ceiling

A capsule declares the bits it needs in its manifest, in `required_caps` and
`optional_caps`. The certificate above the manifest sets `allowed_caps_ceiling`,
the most that identity may ever hold. Verified spawn intersects the two: the
installed set is the manifest request bounded by the certificate ceiling
(`caps::check_ceiling` then `caps::check_grant` in the verify pipeline). A
publisher cannot widen a capsule's authority by editing a manifest, because the
ceiling is fixed in the anchor-signed certificate.

The intersection result is what the kernel installs into the process control
block and mints into the capability token.

---

## Enforcement on the syscall path

Every syscall is gated before its handler runs
(`src/syscall/contract/dispatch.rs:31`):

```
  dispatch(number, args)
      cap = Capability::resolve(number, args)
      if cap is None:
          log the denial
          return EPERM
      invoke(number, args)
```

`resolve` runs a chain of checks (`src/syscall/contract/resolver/resolve.rs`):

```
  check_token            the token's MAC verifies
  check_session_binding  the token belongs to this session
  check_asid_binding     the token belongs to this address space
  check_revocation_epoch the token's epoch is current
  check_syscall_allowed  the held capabilities permit this syscall
```

The last check consults an explicit table that maps each syscall to the
capability it needs (`src/syscall/contract/cap_table/mk.rs`). The table is the
authority; the [ABI reference](../abi/syscalls.md) mirrors it. A representative
slice:

```
  MkMmap                       can_allocate_memory     (Memory)
  MkSpawn, MkIpc*, MkService*  can_ipc                 (IPC)
  MkDeviceClaim, MkDeviceRelease, MkPciConfig*  can_driver  (Driver)
  MkMmioMap, MkMmioUnmap       can_mmio                (Mmio)
  MkIrqBind/Unbind/Poll/Ack    can_irq                 (Irq)
  MkDmaMap, MkDmaUnmap         can_dma                 (Dma)
  MkPioGrant/Read/Write/Release can_pio                (Pio)
  MkSurfaceRegister/Share/Release  can_surface_create  (GraphicsSurfaceCreate)
  MkSurfaceAttach              can_surface_map         (GraphicsSurfaceMap)
  MkSurfacePresent             can_present             (GraphicsPresent)
  MkInputEventPost             can_input_source        (InputSource)
```

A few calls require only a valid token and no specific bit: `MkExit`,
`MkPidAlive`, `MkYield`, the time calls, and `MkCapCheck`. Broker grant calls
add an ownership check on top of the capability: holding `Irq` lets you call
`MkIrqPoll`, but the broker still returns `EPERM` if the grant id you name is not
yours.

---

## The capability token

The token is the proof a capsule carries. It is not a bearer secret that can be
copied between capsules or replayed across boots; it is a structure
authenticated by a keyed MAC and bound to the session, the address space, and the
boot (`src/capabilities/token/types.rs`):

```
  CapabilityToken
    owner_module          u64        the capsule this token belongs to
    permissions           Vec<Capability>
    expires_at_ms         Option<u64>
    nonce                 u64
    signature             [u8; 64]   the keyed MAC over the material below
    token_id              u64
    subject_capsule_id    u32
    subject_asid          u32        the address space it is bound to
    subject_measurement   [u8; 32]   a measurement of the capsule
    boot_session_nonce    [u8; 16]   the boot it was minted in
    revocation_epoch      u64
    delegation_depth      u8         how far it may be re-delegated
```

### Authentication

Verification recomputes the MAC and compares it in constant time
(`src/capabilities/token/verify.rs`):

```
  verify_token(tok)
      key      = the token signing key minted at boot
      material = token_material(tok, bits)       128 bytes, every field above
      computed = mac64(key, material)
      return ct_eq_64(computed, tok.signature)   constant time, no early exit
```

The MAC is two keyed BLAKE3 hashes concatenated to 64 bytes
(`src/capabilities/token/material.rs`):

```
  mac64(key, material)
      mac1 = keyed_blake3(key, material)
      mac2 = keyed_blake3(key, material || "CAP2")
      return mac1 || mac2                         64 bytes
```

The comparison is `ct_eq_64`, which XORs all 64 bytes and folds the difference
to a single branch, so verification time does not depend on how many bytes
matched. That closes the timing side channel an attacker would otherwise use to
forge a signature byte by byte.

### Why it cannot be replayed

The material covers the boot session nonce, so a token minted in one boot fails
to verify in the next: the signing key and the session nonce both change. It
covers the subject address space id, so a token lifted from one capsule does not
authenticate for another. It covers a measurement of the capsule, so the token is
tied to the exact code it was issued to.

### Validity and revocation

A token is valid only if it authenticates, has not expired, and has not been
revoked (`src/capabilities/token/validate.rs`):

```
  is_token_valid(tok) =
      verify_token(tok) and not_expired(tok) and not is_revoked(owner, nonce)
```

Revocation is a set of `(owner_module, nonce)` pairs
(`src/capabilities/token/revocation.rs`), checked on every validation. Revoking
one token, or every token for an owner, takes immediate effect: the next syscall
the affected capsule makes fails the resolver chain and returns `EPERM`.

### Delegation

`delegation_depth` bounds how far a capability can be passed on. `MkCapGrant`
hands a subset of held capabilities to another capsule; the new token's depth is
lower than the granter's, so a delegated capability cannot be re-delegated
without limit. `MkCapRevoke` undoes a grant.

---

## The shape of the guarantee

Putting the pieces together, an action succeeds only if all of the following
hold at once:

```
  the token's MAC verifies under the boot's signing key
  the token is bound to this session, this address space, this boot
  the token has not expired and has not been revoked
  the held capabilities permit this specific syscall
  for a broker call, the caller owns the named grant
```

None of these can be satisfied by a capsule editing its own state, because the
authority rests on a MAC the capsule cannot forge and bindings it cannot fake.
The capability system and the [verified-spawn gate](capsules-and-trust.md) are
the two halves of the same story: spawn decides what a capsule is allowed to
hold, and the token enforces it on every call thereafter.

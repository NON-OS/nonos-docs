# Capabilities and Tokens

A verified capsule is admitted to run, but admission says nothing about what it
may do once running. That is the job of capabilities. Every privileged action in
NØNOS is guarded by a capability bit, and a capsule carries a token that proves
which bits it holds. The kernel checks that token on the way into every system
call, before the call does any work. There is no ambient authority: a capsule
that holds no bits can compute and nothing else.

This page describes the capability set, how a capsule's authority is represented
as an unforgeable token, and exactly where and how the kernel enforces it. It
assumes the [verified-spawn pipeline](capsules-and-trust.md), which decides the
set of bits a capsule is allowed to hold in the first place.

## The capability set

A capability is a single bit. The set is closed and small: thirty-one variants
of the `Capability` enum in `src/capabilities/types/defs.rs:18`. Each variant maps
to one bit through `Capability::bit()` (`src/capabilities/types/bit.rs:20`), and the
bits are the plain powers of two, so a set of capabilities is a `u64` bitmask.

```
  CoreExec               1        DeviceEnum          32768
  IO                     2        Driver              65536
  Network                4        Mmio               131072
  IPC                    8        Irq                262144
  Memory                16        Dma                524288
  Crypto                32        Pio               1048576
  FileSystem            64        InputSource       2097152
  Hardware             128        TimeSet           4194304
  Debug                256        SpawnBroker       8388608
  Admin                512        SpawnWindow      16777216
  RegisterService     1024        ProcessControl   33554432
  GraphicsDisplayQuery   2048     StoreWrite       67108864
  GraphicsSurfaceCreate  4096     EnrolDevRoot    134217728
  GraphicsSurfaceMap     8192     Keyring         268435456
  GraphicsPresent       16384     Entropy         536870912
                                  AppInstall     1073741824
```

The enum is the single source of the bit mapping. No other part of the kernel
writes a raw literal for a capability; grant, revoke, and test all go through the
bitmask algebra in `src/capabilities/bits.rs`, which is nothing more than `OR`,
`AND` with a complement, and `AND` against a single bit. Keeping the whole
authority surface to thirty-one bits is deliberate: it is small enough to audit
in one reading, and every action a capsule performs that reaches beyond its own
address space is one of these bits. The tail of the set is the newer, narrower
authorities: `TimeSet` to set the clock, `SpawnBroker` / `SpawnWindow` /
`ProcessControl` for the process and window managers, `StoreWrite` for the
installer, `EnrolDevRoot` to enrol a signing root (see
[developer roots](developer-roots.md)), `Keyring` to reach the keyring capsule,
`Entropy` to reseed the RNG, and `AppInstall` for the installer. Each is held only
by the one capsule that needs it.

Several bits form structured groups rather than standing alone.

The driver-broker bits are layered, and the enum comment at `types.rs:35` is the
authority for how they compose. `DeviceEnum` permits enumeration only. `Driver`
permits claiming and releasing a device. `Mmio`, `Irq`, `Dma`, and `Pio` are each
required in addition to a claim before the broker will hand over the
corresponding grant: a slice of a device's memory window, an interrupt binding, a
DMA-coherent buffer, or a port-window grant. A capsule holding `Driver` alone can
own a device but touch none of it.

`Admin` is a super-grant over that family. The token predicates that gate the
broker treat `Admin` as satisfying the requirement
(`src/capabilities/token/types/authority_broker.rs:24`, `:31`, `:38`, `:45`, `:52`),
so a capsule with `Admin` passes `can_driver`, `can_mmio`, `can_irq`, `can_dma`,
and `can_pio` without holding those individual bits. `Admin` is itself a capability
that a capsule holds only if its verified manifest was granted it, so this is a
concentration of authority, not a bypass of the check.

Input authority is deliberately split into a producer side and a consumer side, and
the two predicates are not the same set. `can_input_source`
(`authority_broker.rs:59`), the right to *post* an input event, is satisfied by
`InputSource`, `Irq`, or `Admin`, on the reasoning that a capsule already driving an
input device's interrupt is by construction an input source. `can_input_consumer`
(`authority_broker.rs:71`), the right to *drain* the input ring, is satisfied by
`InputSource` or `Admin` but deliberately not by `Irq`: every device driver holds
`Irq`, and accepting `Irq` here would let any driver read every keystroke, which is
exactly a keylogger. Only a capsule explicitly granted `InputSource`, the input
router, may consume the ring (comment at `authority_broker.rs:66-68`).

A driver capsule is the clearest example of a focused grant. The PS/2 input
driver requests exactly the bits it needs (`src/hardware/ps2_kbd_capsule/spawn.rs:51`):

```
  CoreExec | IPC | Memory | Driver | DeviceEnum | Pio | Irq | InputSource
```

It can run, send and receive messages, allocate memory, claim its device,
program its ports and interrupt line, and post input. It holds no `Network`, no
graphics bits, and no authority over any other device. An attempt to act outside
that set returns `EPERM` at the syscall boundary.

## Declaration and the ceiling

A capsule declares the bits it wants in its manifest, split into `required_caps`
and `optional_caps`. The NØNOS identity certificate above the manifest carries an
`allowed_caps_ceiling`, the most that identity may ever hold, and the certificate
is signed by the trust anchor. Verified spawn bounds the manifest request by that
ceiling: `caps::check_ceiling` rejects a manifest asking for anything above the
ceiling, and `caps::check_grant` computes the installed set. A publisher cannot
widen a capsule's authority by editing a manifest, because the ceiling is fixed
in the anchor-signed certificate, not in the manifest. The [verified spawn
page](capsules-and-trust.md) covers that pipeline; the result is a `u64` of bits
installed into the process control block and minted into a token.

## The capability token

The token is the proof a capsule carries at runtime. It is not a bearer secret
that can be copied between capsules or replayed across boots. It is a structure
authenticated by a keyed MAC and bound to a boot and an address space
(`src/capabilities/token/types/defs.rs:23`):

```
  CapabilityToken
    owner_module          u64             the capsule the token belongs to
    permissions           Vec<Capability> the granted capabilities
    expires_at_ms         Option<u64>     optional wall-clock expiry
    nonce                 u64             per-owner value, the revocation key
    signature             [u8; 64]        the keyed MAC over everything else
    token_id              u64             a unique id for this mint
    subject_capsule_id    u32             the pid the token is bound to
    subject_asid          u32             the address space it is bound to
    subject_measurement   [u8; 32]        reserved, currently zero
    boot_session_nonce    [u8; 16]        the boot it was minted in
    revocation_epoch      u64             the capsule's revocation counter
    delegation_depth      u8              depth in a delegation chain
```

One field carries less than its name suggests, and the documentation says so
plainly. `subject_measurement` is thirty-two bytes of zero in the current kernel.
It is populated on every mint and covered by the MAC (`material.rs:34`), so a
future measurement binding can be turned on without changing the token format,
but nothing today computes a capsule measurement or checks one. The token is not
bound to a code measurement. It is bound to the boot and the address space, and
those bindings are real; the measurement binding is not yet.

### Authentication

Verification recomputes the MAC and compares it in constant time
(`src/capabilities/token/verify.rs:24`). It is short enough to state in full:

```
  verify_token(tok):
      key = signing_key()            minted at boot; if unset, return false
      material = token_material(tok, caps_to_bits(tok.permissions))
      computed = mac64(key, material)
      return ct_eq_64(computed, tok.signature)
```

The first line is a fail-closed guard: on a kernel where the signing key has not
been set, verification returns `false` rather than proceeding. The bitmask fed
into the material is derived from the `permissions` vector at that moment by
`caps_to_bits`, not read from a stored field, so the vector and the bits it
implies cannot disagree without breaking the signature.

The material is a fixed 128-byte buffer with a field at every offset
(`src/capabilities/token/material.rs:25`):

```
  0..8     owner_module        48..80   subject_measurement
  8..16    capability bitmask  80..96   boot_session_nonce
  16..24   expires_at_ms       96..104  revocation_epoch
  24..32   nonce               104      delegation_depth
  32..40   token_id            105..128 zero padding
  40..44   subject_capsule_id
  44..48   subject_asid
```

The authenticator is `mac64` (`material.rs:41`): two keyed BLAKE3 hashes of the
same material under the same key, concatenated to sixty-four bytes. The second
hash absorbs the literal suffix `CAP2` before finalising, which separates the two
halves so they are independent outputs rather than one 256-bit hash written
twice. The comparison uses `ct_eq_64` from `crypto/util/constant_time`, which
folds the difference over all sixty-four bytes without an early exit, so
verification time does not reveal how many bytes matched. That closes the timing
channel an attacker would otherwise walk to forge a tag one byte at a time.

### Why it cannot be replayed or transplanted

The material covers `boot_session_nonce`, and the signing key is minted fresh at
each boot, so a token from one boot fails to verify in the next: different key
over a different nonce. The material covers `subject_asid`, so a token lifted out
of one capsule does not authenticate for another, which runs in a different
address space. These two bindings are what make the token safe to hold in
userspace: possession of the bytes is not possession of the authority.

### Validity and revocation

A token is valid only if it authenticates, has not expired, and has not been
revoked (`src/capabilities/token/validate.rs`):

```
  is_token_valid(tok) =
      verify_token(tok) and not_expired(tok) and not is_revoked(owner, nonce)
```

Authority is withdrawn two ways. The direct way is a set of `(owner_module,
nonce)` pairs (`src/capabilities/token/revocation.rs`), consulted on every
validation; revoking a nonce, or every nonce for an owner, takes effect on the
capsule's next syscall. The scalable way is the revocation epoch. Rather than
hunt down every outstanding copy of a token when a capsule's authority changes,
the kernel bumps a per-process revocation counter and mints the capsule a fresh
token carrying the new epoch. Older tokens still authenticate, but they carry an
epoch behind the process's current one, and the resolver rejects them. One
counter increment retires every token minted before it.

`delegation_depth` bounds how far a capability can be passed on. `MkCapGrant` and
`MkCapRevoke` are the syscalls that hand a subset of held capabilities to another
capsule and undo it; both are gated by the `IPC` capability (see the table
below). The delegation machinery lives in `src/capabilities/delegation/`.

## Enforcement on the syscall path

Every syscall is gated before its handler runs (`src/syscall/contract/dispatch.rs:31`).
The contract resolves the calling capsule's token and runs it through a fixed,
ordered chain (`src/syscall/contract/resolver/resolve.rs:31`):

```
  check_token             the MAC verifies, the token is unexpired and unrevoked
  check_session_binding   the token's boot nonce is this boot's nonce
  check_asid_binding      the token's address space id is the caller's
  check_revocation_epoch  the token's epoch is not behind the process epoch
  check_syscall_allowed   the held bits permit this specific syscall
```

The order is not incidental. Authenticity is established first, so every field a
later step reads is already known genuine. The context bindings come next, so a
genuine but stale or transplanted token is rejected before the per-syscall test.
Only then does the chain consult the bits. Any failure turns the dispatch into
`EPERM`, logs the denial, and the handler never runs.

The final check consults an explicit table mapping each syscall to the authority
it needs (`src/syscall/contract/cap_table/mk.rs`). The table is the authority;
the [ABI reference](../abi/syscalls.md) mirrors it. Read exactly as the code
stands:

```
  valid token, no specific bit
    MkExit, MkPidAlive, MkYield, MkTimeMillis, MkTimeRtc, MkBatteryStatus,
    MkProcStat, MkAttestStatus, MkCapCheck

  Memory        MkMmap (allocate), MkMunmap (deallocate)
  IPC           MkSpawn, all MkIpc*, MkServiceLookup/Register,
                MkCapGrant, MkCapRevoke, MkThreadSpawn, MkProcOutput
  InputSource   MkInputEventPost (can_input_source: InputSource|Irq|Admin, mk.rs:99)
                MkInputEventDrain, MkInputEventWait (can_input_consumer: InputSource|Admin, mk.rs:106)
  CoreExec      MkGetPid, MkArgs  (via can_getpid)
  CoreExec+IPC+Memory  MkCapsuleLoad  (all three required)

  DeviceEnum    MkDeviceList
  Driver        MkDeviceClaim, MkDeviceRelease, MkPciConfigRead/Write
  Mmio          MkMmioMap, MkMmioUnmap
  Irq           MkIrqBind, MkIrqUnbind, MkIrqAck, MkIrqPoll, MkIrqWait
  Dma           MkDmaMap, MkDmaUnmap
  Pio           MkPioGrant, MkPioRead, MkPioWrite, MkPioRelease
  Debug         MkDebug

  GraphicsSurfaceCreate  MkSurfaceRegister, MkSurfaceShare, MkSurfaceRelease
  GraphicsSurfaceMap     MkSurfaceAttach
  GraphicsPresent        MkSurfacePresent
  GraphicsDisplayQuery   MkDisplayVsyncWait
```

Two details of the table are worth stating because they are easy to assume
wrong. Draining and waiting on the input ring (`MkInputEventDrain`,
`MkInputEventWait`) require `can_input_consumer` (`mk.rs:106-107`), which is
`InputSource` or `Admin` but pointedly not `Irq`, while posting an event
(`MkInputEventPost`, `mk.rs:99`) requires `can_input_source`, which does accept
`Irq`. The split is the anti-keylogging boundary described above: any driver can
generate input off its own interrupt, but only the router may read the ring. And a
syscall the `Mk` table does not name at all falls through with `_ => return None`
(`mk.rs:118`), which lets another capability family claim it or, failing that,
denies it.

The broker calls layer one more check on top of the capability. Holding `Irq`
lets a capsule call `MkIrqPoll`, but the broker still returns `EPERM` if the
grant id it names is not one the capsule owns. The capability is permission to
participate; ownership of the specific grant is checked separately, on the
[hardware broker](../subsystems/hardware-broker/README.md) page.

## The shape of the guarantee

An action succeeds only if all of the following hold at once:

```
  the token's MAC verifies under this boot's signing key
  the token is bound to this session, this address space, and this boot
  the token has not expired and has not been revoked
  the held capabilities permit this specific syscall
  for a broker call, the caller owns the named grant
```

None of these can be satisfied by a capsule editing its own state, because the
authority rests on a MAC the capsule cannot forge and bindings it cannot fake.
The capability system and the [verified-spawn gate](capsules-and-trust.md) are
the two halves of one story: spawn decides what a capsule is allowed to hold, and
the token enforces it on every call thereafter.

## Debugging a capability denial

A capsule that is running but has a syscall refused is failing the resolver, and
the marker is `[CAP-DENY]`. `dispatch` (`src/syscall/contract/dispatch.rs:31`)
calls `Capability::resolve`, and on any failure it logs
`[CAP-DENY] pid=<pid> syscall=<name>(<number>)` (`dispatch.rs:45`) and returns
`EPERM`, which is `1` (`src/syscall/types/errnos/posix.rs:18`). So a userspace
call that comes back as `-EPERM` with a matching `[CAP-DENY]` line naming the
syscall is a capability or token-binding refusal, and the syscall name in the log
tells you which call was denied.

The log does not tell you which of the five resolver steps failed, and that is
worth knowing when reading it. `Capability::resolve`
(`src/syscall/contract/capability.rs:44`) runs the ordered chain and collapses any
`ResolverError` into `None` with `.ok()?`, so the `[CAP-DENY]` line records the
syscall but not the reason. The underlying `ResolverError`
(`src/syscall/contract/resolver/error.rs:18`) distinguishes eight causes,
`TokenSignatureInvalid`, `TokenExpired`, `TokenRevoked`, `BootSessionNotLatched`,
`BootSessionMismatch`, `AsidMismatch`, `RevocationEpochStale`, and
`SyscallNotPermitted`, but only the first of the five checks that a step names is
surfaced past the `.ok()`. To reason about which one fired, walk the fixed order:
a token that fails at all after a re-mint is `RevocationEpochStale`, a token that
worked last boot is `BootSessionMismatch`, and a syscall the held bits simply do
not cover is `SyscallNotPermitted`, the last check.

To decode which bit a syscall needs, read the `Mk` table
(`src/syscall/contract/cap_table/mk.rs`) reproduced above: `MkMmioMap` needs
`Mmio`, `MkIrqBind` needs `Irq`, `MkDeviceClaim` needs `Driver`, and so on. Then
compare against the bits the capsule actually holds, which are the powers of two in
`src/capabilities/types/bit.rs` decoded by `bits_to_caps` (`src/capabilities/bits.rs:29`)
from the token's `permissions`. A `[CAP-DENY]` on `MkIrqBind` from a capsule whose
grant did not include `Irq` (`262144`) is a manifest problem, not a runtime bug:
the capsule was admitted without the bit it is now trying to use, and the fix is in
its declared `required_caps`, not in the kernel. A separate case is a broker call
that passes the capability check but names a grant the capsule does not own; that
`EPERM` comes from the broker, not the resolver, and there is no `[CAP-DENY]` for
it, as the [hardware broker](../subsystems/hardware-broker/README.md) page covers.

One more distinction, because it is the common confusion: a capsule that never
starts at all is not a `[CAP-DENY]`. A denial is a running capsule losing one
syscall. A capsule that fails to spawn because its requested caps exceed the
manifest or certificate ceiling fails much earlier, at
[verified spawn](capsules-and-trust.md), as a
`SpawnError::ManifestRejected(ManifestVerifyError::CapsExceedCeiling)` or
`GrantOutsideManifest`, not as a runtime `EPERM`.

## Source map

```
  src/capabilities/types/defs.rs        the Capability enum (31 variants)
  src/capabilities/types/bit.rs         Capability::bit(), the power-of-two mapping
  src/capabilities/bits.rs              caps_to_bits, bits_to_caps, the bitmask algebra
  src/capabilities/token/types/defs.rs  the CapabilityToken struct
  src/capabilities/token/types/authority_broker.rs  can_driver/mmio/irq/dma/pio, can_input_source/consumer
  src/capabilities/token/material.rs    the 128-byte MAC material and mac64
  src/capabilities/token/verify.rs      verify_token
  src/capabilities/token/validate.rs    is_token_valid
  src/capabilities/token/revocation.rs  the revoked (owner, nonce) set
  src/capabilities/delegation/          delegation and its depth bound
  src/syscall/contract/dispatch.rs      the pre-handler gate and the [CAP-DENY] log
  src/syscall/contract/capability.rs    Capability::resolve, where ResolverError becomes None
  src/syscall/contract/resolver/        the ordered resolve chain
  src/syscall/contract/resolver/error.rs the eight ResolverError variants
  src/syscall/contract/cap_table/mk.rs  the syscall-to-capability table
  src/syscall/types/errnos/posix.rs     EPERM = 1
```

The spawn side that decides the bits a capsule may hold, and its own
`SpawnError` variants, are on the
[verified spawn](capsules-and-trust.md) page; the signing key and MAC underneath
the token are on the [signing and MAC](signing-and-mac.md) page.

# PIO Grants

Some x86 devices are driven through port-mapped I/O rather than memory-mapped registers. The
broker grants a capsule a specific port window and then performs the `in` and `out` instructions
on its behalf, checking every access against the grant. Port I/O is an x86-only instruction
class, so this whole path is compiled only on x86_64; other architectures fail the syscalls
closed with `ENOSYS`. This page documents `MkPioGrant` and the checked accesses. The code is
under `src/hardware/broker/pio/`.

## The grant

A `PioGrant` (`src/hardware/broker/pio/grant.rs:34`) records a contiguous port window for a
holder:

```
  struct PioGrant {
      grant_id: u64, pid: u32, device_id: u64, claim_epoch: u64,
      port_base: u16, port_count: u16,
  }
```

The grant is issued from `grant_for_caller` after the same claim and epoch check the other grant
classes run, and it is recorded in the global PIO grant table. The window is `[port_base,
port_base + port_count)`, and it is the exact set of ports the capsule is allowed to touch.

## Checked access

The capsule does not execute `in` or `out` itself, because that would require the kernel to hand
it I/O privilege over the whole port space. Instead it calls `MkPioRead` and `MkPioWrite`, and
each access is resolved against the grant table (`pio/grant.rs:54`, `pio/access/resolve.rs`):

```
  lookup_for_holder(pid, grant_id):
      g = grant with this grant_id     else UnknownGrant
      if g.pid != pid:                 NotHolder
      return g
```

The access is then bounds-checked against the grant's port window and the requested width before
the kernel issues the instruction. A missing grant, a grant held by another pid, or a port
outside the granted window is refused. This is the same shape as the MMIO story: the kernel holds
the privileged capability (here, I/O port access) and mediates every use of it against a
per-capsule grant, so a capsule can drive its device's ports and only its device's ports.

## Non-x86 builds

Because port I/O does not exist off x86, the broker's `pio` submodule is gated on
`target_arch = "x86_64"` (`src/hardware/broker/mod.rs:32`), and on other architectures the
syscall layer fail-closes the PIO calls with `ENOSYS` through
`syscall/microkernel/pio/unsupported.rs`. The behavior is explicit rather than a silent no-op: a
capsule that asks for port I/O on a platform without it gets a clear unsupported error.

## Revocation

The PIO grant table is drained on `MkPioRelease` (single grant), on `MkDeviceRelease` (every
grant on the device), and on capsule exit (`release_all_for_pid`), the same three-way revocation
the other classes use; `remove` and the `drain_*` helpers all enforce holder ownership. See
[revocation](revocation.md).

## Source

```
  src/hardware/broker/pio/grant.rs           the PioGrant table and holder checks
  src/hardware/broker/pio/access/resolve.rs  per-access bounds checking
  src/hardware/broker/mod.rs                 the x86-only cfg gate
```

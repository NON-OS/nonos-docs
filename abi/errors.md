# Error Codes

Every syscall returns an `i64`. A non-negative value is success. A negative value
is an error, and its magnitude is a POSIX errno number. This page lists the codes
the kernel returns, how they are encoded, and what causes the common ones. It is
the companion to the [syscall reference](syscalls.md).

---

## Return convention

A handler produces a `SyscallResult` whose `value` field is the `i64` returned
(`src/syscall/types/result.rs`):

```
  SyscallResult::error(errno) = value of -(errno)
  is_error()                  = value < 0
```

The entry stub casts that `i64` into `RAX` as a `u64`
(`src/arch/x86_64/syscall/manager/entry.rs:22`), so a negative result reaches
userspace as a large unsigned value that a client reinterprets as a signed
negative. The rule for a caller is simple: treat the return as `i64`, and any
value below zero is an error whose magnitude is the errno below.

```
  return >= 0    success: a length, a pid, a handle, a count, or zero
  return <  0    error:   -errno
```

## The codes

The microkernel error constants are defined as negative `i64` values
(`src/syscall/microkernel/errnos.rs:22`). Their magnitudes are the standard POSIX
numbers:

| Constant | Value | POSIX | Meaning in NØNOS |
|----------|-------|-------|------------------|
| ERRNO_PERM | -1 | EPERM | Capability denied, or not the owner of a broker grant. |
| ERRNO_NOENT | -2 | ENOENT | No such inbox, service, or object. |
| ERRNO_NOMEM | -12 | ENOMEM | A ring or table was full, or an allocation failed. |
| ERRNO_ACCES | -13 | EACCES | Access refused by policy. |
| ERRNO_FAULT | -14 | EFAULT | A user pointer argument was not readable or writable. |
| ERRNO_BUSY | -16 | EBUSY | The resource is in use, for example a device already claimed. |
| ERRNO_NODEV | -19 | ENODEV | No such device. |
| ERRNO_INVAL | -22 | EINVAL | An argument was out of range or malformed. |
| ERRNO_NOSYS | -38 | ENOSYS | Unknown syscall number, or a number with no handler. |
| ERRNO_NOTSUP | -95 | EOPNOTSUPP | The operation is not supported in this state. |
| ERRNO_TIMEDOUT | -110 | ETIMEDOUT | A blocking call reached its deadline with no event. |
| ERRNO_STALE | -116 | ESTALE | A handle refers to a freed or reused object, for example a stale surface handle. |

## What causes the common ones

### EPERM (-1)

The capability gate denied the call. It is returned from the dispatcher when the
resolver chain fails (`src/syscall/contract/dispatch.rs:36`): the token did not
hold the required capability, the token failed to authenticate, or its session,
address space, or revocation binding did not hold. For a broker grant call it is
also returned when the caller does not own the grant it named, even with the
right capability bit. When a driver gets `EPERM` on `MkIrqPoll`, check both that
the manifest declared `Irq` and that the poll names the grant the bind returned.

### EFAULT (-14)

A user pointer was bad. Every syscall that reads or writes user memory validates
the pointer first (`src/usercopy`). `validate_user_read` and
`validate_user_write` walk the range and confirm each page is mapped, user
accessible, and writable where needed; any failure becomes `EFAULT`. The typed
accessors `read_user_value` and `write_user_value` additionally require the
pointer to be aligned for the type, and a misaligned pointer is also `EFAULT`. A
null pointer where one is required is `EFAULT` as well.

### ENOENT (-2)

A name did not resolve. `MkServiceLookup` returns it when the service is not in
the registry (`src/syscall/microkernel/ipc/lookup.rs`); `MkIpcRecv` returns it
when the target inbox does not exist (`src/syscall/microkernel/ipc/recv.rs`).

### ETIMEDOUT (-110)

A blocking call ran out of time. `MkIpcRecv` and `MkIpcCall` return it when the
timeout elapses with no message (`src/syscall/microkernel/ipc/recv.rs`). A
`timeout_ms` of zero on `MkIpcCall` is not "no timeout"; it selects the default of
five seconds.

### ENOMEM (-12)

A bounded resource was exhausted. `MkMmap` returns it when virtual address space
allocation fails (`src/syscall/microkernel/memory/mmap.rs`); inbox and endpoint
registration return it when their tables are full. The input ring drops events
rather than returning this; the others are hard limits.

### EINVAL (-22)

An argument was malformed: a zero-length buffer where one is required, an unknown
device id, an out-of-range value. It is the catch-all for a request the kernel can
parse but will not act on.

---

## A note on layering

There are two error spellings in the tree. The microkernel handlers use the
negative `ERRNO_*` constants above and return them directly. The contract layer
uses positive POSIX names and `SyscallResult::error`, which negates them. Both
produce the same thing in `RAX`: a negative `i64` whose magnitude is the POSIX
number. From a capsule's point of view there is one convention, the table above,
and the internal spelling does not matter.

# The User/Kernel Copy Boundary

When the kernel reads or writes a capsule's memory, syscall arguments, an IPC
payload, a buffer a driver passed down, it never dereferences the user pointer
directly. It validates the range against a pure policy, walks the page tables to
confirm every page is present with the right permission, and then transfers the
bytes through the physical direct map, with interrupts disabled so the mapping the
walk approved cannot change before the copy uses it. This is the surface the
[Kani proofs](../../../verification/README.md) exercise for panic-freedom and
bounds. The module is layered so that each concern lives in one file, and the code
is under `src/usercopy/`.

Dereferencing a raw user pointer in kernel mode is the classic way a kernel is
compromised. The pointer could be null, unmapped, non-canonical, or aimed at the
kernel's own address range, and following it either faults in kernel context or
turns the kernel into a confused deputy writing where a capsule could not. The
boundary here removes the raw dereference entirely: the user virtual address is a
value to be validated, and the actual memory access goes through the kernel's
direct map at the physical frame the page tables resolve to.

## The range policy

The innermost layer is a pure function over an address and a length, with no page
walking and no permission decision (`src/usercopy/policy.rs:35`):

```
  USER_SPACE_END = 0x0000_7FFF_FFFF_FFFF
  MAX_COPY_SIZE  = 64 MiB

  check_range(addr, len):
      if addr == 0                       -> NullPointer
      if len > MAX_COPY_SIZE              -> SizeTooLarge
      if len == 0                        -> Ok(None)
      end = addr.checked_add(len - 1)    -> AddressOverflow on wrap
      if end > USER_SPACE_END             -> InvalidAddress
      Ok(Some(UserRange { start_page, end_page }))    aligned down to pages
```

Five things are decided here and nowhere else, so every caller errors the same way:
a null base is rejected, a length above the 64 MiB cap is rejected, a zero length is
a successful no-op, an address plus length that overflows a `u64` is rejected before
it can wrap, and a range whose last byte lies above the canonical user limit is
rejected. That last check is the one that keeps a user copy inside user space: the
end of the range must be at or below `USER_SPACE_END`, so a buffer that would run off
the top of user memory into the non-canonical gap or the kernel half never passes.
The function returns the page-aligned range for the next layer to walk, or `None`
when there is nothing to copy.

## The per-page walk

The next layer applies the range policy and then walks the pages
(`src/usercopy/validate.rs:34`):

```
  validate(addr, len, need_write):
      range = check_range(addr, len)?     None -> Ok, nothing to copy
      for each page from range.start_page to range.end_page:
          if need_write: translate_write(page)?
          else:          translate_read(page)?
```

`validate_user_read` calls this with `need_write = false` and `validate_user_write`
with `true`. For every page the range touches, it asks the walker
(`src/usercopy/walk/`) to resolve the page and confirm it is present and carries the
required permission: `translate_read` for a read, `translate_write` for a write. A
page that is not present, or is present but lacks the needed permission, fails the
walk and the whole validation fails. The page loop advances with a saturating add and
breaks if the address wraps to zero, so it terminates even at the very top of the
address space. After `validate` returns `Ok`, every page of the range is known to be
a present, correctly-permissioned user page.

## The copy

The outer layer validates and then transfers, and it never touches the user pointer
as a pointer (`src/usercopy/copy.rs:27`):

```
  copy_from_user(user_ptr, dst):
      run_without_interrupts(||
          validate_user_read(user_ptr, dst.len())?
          copy_from_user_directmap(user_ptr, dst))

  copy_to_user(user_ptr, src):
      run_without_interrupts(||
          validate_user_write(user_ptr, src.len())?
          copy_to_user_directmap(user_ptr, src))
```

The transfer goes through `copy_from_user_directmap` and `copy_to_user_directmap`
(`src/usercopy/direct.rs`), which the module documentation states copy through the
direct map after validation has cleared the range and access policy, so the user
virtual address is never dereferenced. The kernel reaches the user's bytes at the
physical frame the walk resolved, through the direct map, not by following the user
pointer in the current address space.

## Why interrupts are disabled

The whole validate-then-copy sequence runs inside `run_without_interrupts`. That is
what closes the time-of-check-to-time-of-use gap. If a timer interrupt could fire
between the page walk and the copy, the faulting process could be preempted and its
mappings changed, so the page the walk approved might not be the page the copy
touches. Holding interrupts off across both steps means the mapping validated is the
mapping used. It also keeps the copy from being interrupted partway and re-entering
the paging paths, matching the interrupt discipline the
[paging manager](paging-manager.md) uses.

## Errors and the wider module

The error type is `UsercopyError` (`src/usercopy/error.rs`): the range policy returns
`NullPointer`, `SizeTooLarge`, `AddressOverflow`, and `InvalidAddress`, and the walk
adds its presence and permission failures. Around the byte-slice copies documented
here, the module also provides typed value copies, string copies with the same range
rules (`src/usercopy/string.rs`), and the low-level direct-map helpers, all built on
the same `check_range` and `walk` foundation so they all agree on what a valid user
range is.

## Verification

This boundary is one of the surfaces the verification stack proves rather than just
tests. The Kani harnesses in the kernel proof crates check that the validation and
copy paths are panic-free and undefined-behaviour-free over bounded inputs, and the
runnable proofs check the range invariants, so the guarantee that a user copy stays
within user space and never dereferences an unvalidated pointer is machine-checked,
not only asserted here.

## Source

```
  src/usercopy/policy.rs    check_range, the pure range rules and the limits
  src/usercopy/validate.rs   validate_user_read / validate_user_write, the page walk
  src/usercopy/copy.rs       copy_from_user / copy_to_user
  src/usercopy/direct.rs     the direct-map transfer helpers
  src/usercopy/walk/         the page-table walk and permission decision
  src/usercopy/error.rs      UsercopyError
```

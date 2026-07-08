# The Syscall Boundary

A syscall crosses from a capsule in ring 3 into the kernel in ring 0, is decoded to a
typed number, passes the capability contract, and dispatches to a handler. What makes
the NØNOS boundary notable is that the capability check is not merely a runtime gate that
a handler could forget to call; it is a type-enforced precondition, so a handler cannot
run without proof that the check happened. This page documents the entry, the register
ABI, the number decode, the contract gate, and the witness type that enforces the check.
The code is under `src/arch/x86_64/syscall/` and `src/syscall/`.

## The instruction and the stub

On x86_64 the boundary is the `SYSCALL` instruction. `LSTAR` is programmed during core
init to point at an assembly entry stub, which saves the user state, switches to the
kernel `GS` and stack, translates the user register layout into the System V calling
convention, and calls `syscall_handler` (`src/arch/x86_64/syscall/manager/entry.rs:22`):

```
  #[no_mangle]
  syscall_handler(number, arg1..arg6) -> u64:
      sc = SyscallNumber::from_u64(number), else return -ENOSYS
      result = contract_dispatch(sc, SyscallArgs::new([arg1..arg6]))
      result.value as u64
```

It is `extern "C"` and `#[no_mangle]` because the assembly stub calls it by name. It is
the arch-specific bridge; everything past `contract_dispatch` is arch-neutral.

## The register ABI

Arguments follow the System V register order, with one substitution the `SYSCALL`
instruction forces:

```
  a0 -> RDI    a1 -> RSI    a2 -> RDX
  a3 -> R10    a4 -> R8     a5 -> R9
  syscall number -> RAX     return value -> RAX
```

`R10` stands in for `RCX` because `SYSCALL` clobbers `RCX` with the return address, so a
capsule places the fourth argument in `R10` and the stub moves it into place. The number
travels in `RAX` and the return value comes back in `RAX`, with errors returned as a
negative errno, which is why an unknown number returns `-ENOSYS` cast to `u64`.

## Number decode

The raw `u64` is decoded to a `SyscallNumber` (`src/syscall/numbers/`) before anything
else, and a value that does not correspond to a known syscall is rejected immediately
with `ENOSYS` rather than reaching the contract. The number scheme itself, four-character
ASCII tags packed into a word so they read as mnemonics in a trace, is documented on the
[numbers](numbers.md) page. Only a valid, typed `SyscallNumber` proceeds.

## The contract gate

`contract_dispatch` (`src/syscall/contract/dispatch.rs:31`) is the single gate every
syscall passes, and its doc comment states that there is no other path that runs the
capability resolution:

```
  dispatch(number, args):
      cap = Capability::resolve(number, args)
      if cap is None:
          log the denial
          return EPERM
      invoke(number, args)      the router
```

It resolves the calling thread's capability against the requested syscall and its
arguments. If resolution fails, it logs a `CAP-DENY` with the pid and syscall and returns
`EPERM`, and the handler never runs. If it succeeds, it invokes the router with the raw
arguments. This is the one place `Capability::resolve` is called from, so every syscall in
the system goes through exactly this check.

## The capability witness

The mechanism that makes the check unforgettable is the `Capability` type
(`src/syscall/contract/capability.rs:30`):

```
  pub struct Capability { token: CapabilityToken }    the field is private

  Capability::resolve(number, args) -> Option<Capability>:
      proc = current process, else None
      ctx = ResolveContext { current_asid, boot_session_nonce, capsule_revocation_epoch }
      resolver_resolve(proc.token, number, args, ctx).ok()?
      Some(Capability { token })
```

The struct wraps a capability token behind a private field, and its only constructor is
`resolve`. User-space cannot build one, and in-kernel code outside the contract module
cannot either, because the field is private. As the source puts it, a handler that takes a
`Capability` therefore has executable proof that the check ran: the check is encoded in
the type, not left to a convention a caller might skip. `resolve` builds the resolve
context, the address-space id, the boot-session nonce, and the capsule's revocation
epoch, and runs the resolver chain over the process's token; only if that chain passes
does a `Capability` come into existence.

## The resolver chain

The five checks `resolve` runs, that the token's MAC verifies and it is unexpired and
unrevoked, that it is bound to this boot and this address space, that its revocation epoch
is current, and that the held bits permit this specific syscall, are the ordered resolve
chain documented in full on the [capabilities page](../../security/capabilities-and-tokens.md).
The boundary here is where that chain is invoked; the capability model is where it is
defined. On success the router runs; the [router](router.md) page covers the dispatch to
the per-family handlers.

## Source

```
  src/arch/x86_64/syscall/manager/entry.rs   syscall_handler, the arch bridge
  src/arch/x86_64/syscall/manager/init.rs     LSTAR/STAR/SFMASK setup
  src/syscall/contract/dispatch.rs             the contract gate
  src/syscall/contract/capability.rs           the Capability witness and resolve
  src/syscall/contract/resolver/               the ordered resolve chain
  src/syscall/numbers/                         SyscallNumber and from_u64
```

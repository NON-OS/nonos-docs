# Syscall Numbers

Syscall numbers in NØNOS are not opaque integers; they are four-character ASCII tags
packed into a word, so that a syscall reads as a mnemonic in a register dump or a trace.
This page documents the tag encoding, how a raw number is decoded to a typed syscall, and
the families the numbers fall into. The code is under `src/syscall/abi/` and
`src/syscall/numbers/`.

## The tag encoding

A tag is four ASCII bytes packed little-endian into a `u64` (`src/syscall/abi/tag.rs:20`):

```
  tag4(b) = b[0] | (b[1] << 8) | (b[2] << 16) | (b[3] << 24)
```

The first byte occupies the lowest eight bits, so `tag4(b"MDBG")` is a `u64` whose bytes
read back as `MDBG` when dumped at low memory. That is the whole point: a capsule that
invoked `MDBG` shows `MDBG` in a trace rather than a number a reader would have to look
up. The tags are constructed with this `const fn`, so the numbering lives in one place and
cannot drift.

## Decoding

The raw `u64` from `RAX` is turned into a typed `SyscallNumber` by `from_u64`
(`src/syscall/numbers/convert.rs:22`), which is a lookup:

```
  SyscallNumber::from_u64(id) = abi::lookup_id(id)
```

`lookup_id` maps a known tag to its `SyscallNumber` enum variant and returns `None` for a
value that is not a registered syscall. A `None` is what the [boundary](boundary.md)
turns into `ENOSYS`, so only a tag that decodes to a real syscall proceeds past the
entry. The enum is the authoritative registry; the tag is its wire encoding.

## The families

The tags group by their leading letters into families, and the [router](router.md)
dispatches on them:

```
  Crypto*    random, hash, encrypt and decrypt (with and without AAD),
             ed25519 verify, x25519 public and shared, hmac-sha256,
             hkdf-sha256, keccak256, secp256k1 sign and pubkey
  Admin*     reboot, shutdown, policy push
  Graphics*  display dimensions
  Mk*        the microkernel surface: ipc, memory, spawn and exit, threads,
             time, capabilities (grant, revoke, check), device claim,
             mmio, irq, dma, pci config, pio, surfaces, and input events
```

The `Mk` family is the microkernel proper, the calls that only the kernel can perform:
message passing, memory mapping, process spawn and exit, the hardware broker grants, and
the surface and input paths. The `Crypto` family is the in-kernel cryptographic
primitives, and `Admin` is the small set of privileged control operations. Each family
maps to a required capability, documented on the
[capabilities page](../../security/capabilities-and-tokens.md).

## The exhaustive reference

This page is the shape of the numbering. The exhaustive per-call contract, every syscall
with its number, arguments, required capability, and error codes, is the
[ABI reference](../../abi/syscalls.md), which is generated to match the enum so the two
cannot disagree.

## Source

```
  src/syscall/abi/tag.rs        tag4, the four-character encoding
  src/syscall/abi/              lookup_id and the registry
  src/syscall/numbers/defs.rs   the SyscallNumber enum
  src/syscall/numbers/convert.rs from_u64
```

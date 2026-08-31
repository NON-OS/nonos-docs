# Primitives: SHA-256, SHA-512, HMAC-SHA512, PBKDF2

Everything in the derivation tree is built from a handful of hash and
key-derivation primitives, and `nonos_hd` carries its own because there is no
system crypto library to lean on and because a wallet's seed math has to be
exactly the standard, bit for bit. This page is the primitive layer. It is
`no_std` and allocation-free: every function works in fixed stack buffers, which
matters because these buffers hold pre-image material for private keys.

## SHA-256

`sha256` (`userland/nonos_hd/src/sha256.rs`) is a straight FIPS 180-4 SHA-256
over a byte slice, returning a 32-byte digest. It is used in two places in this
crate: the BIP39 checksum (the first `entropy_bits / 32` bits of the digest are
appended to the entropy before it is split into words) and anywhere a 256-bit
hash is needed. There is nothing exotic here; it exists so the checksum and the
mnemonic round-trip do not depend on an external hash.

## SHA-512 and HMAC-SHA512

`sha512` and `Sha512` (`userland/nonos_hd/src/sha512.rs`) are FIPS 180-4 SHA-512,
exposed both as a one-shot (`sha512`) and as an incremental state (`Sha512`) so a
message can be fed in pieces without allocating a contiguous buffer for it.

`hmac_sha512` and `HmacSha512` (`userland/nonos_hd/src/hmac512.rs`) are
HMAC-SHA512 on top of it, again one-shot and incremental. HMAC-SHA512 is the
workhorse of BIP32: the master key is `HMAC-SHA512("Bitcoin seed", seed)` and
every child derivation is another HMAC-SHA512 keyed by the parent chain code.
The incremental `HmacSha512` is what lets a child derivation stream the tag
`0x00 || parent_key || ser32(index)` through the MAC without building that
concatenation in one allocation.

## PBKDF2-HMAC-SHA512

`pbkdf2_hmac_sha512` (`userland/nonos_hd/src/pbkdf2.rs`) is RFC 2898 PBKDF2 with
HMAC-SHA512 as the PRF, and it is specialised to exactly what BIP39 needs rather
than being general. BIP39 asks for a 64-byte seed, and HMAC-SHA512 already
produces 512 bits, so the output is a single PRF block and there is no
block-index loop:

```
F(1) = U1 xor U2 xor ... xor Uc
U1   = PRF(password, salt || 0x00000001)
Ui   = PRF(password, U(i-1))
```

The implementation computes `U1`, copies it into the output, then for each of the
remaining `iterations - 1` rounds computes the next `U`, wipes the previous one,
and folds it into the output by XOR. BIP39 fixes `iterations` at 2048, and this
code runs the real 2048, not a reduced count. Two properties are worth stating
because they are the reason this is written by hand rather than pulled in:

- **Allocation-free.** All state is a pair of 64-byte stack buffers.
- **Wiped.** Every intermediate `U` is zeroized the instant it is folded in, and
  the last one is wiped before return, so no round's PRF output outlives the
  call. The seed it produces is the caller's to protect from there.

## API summary

| Function | Source | Output |
|---|---|---|
| `sha256(msg)` | `src/sha256.rs` | `[u8; 32]` |
| `sha512(msg)` / `Sha512` | `src/sha512.rs` | `[u8; 64]` |
| `hmac_sha512(key, msg)` / `HmacSha512` | `src/hmac512.rs` | `[u8; 64]` |
| `pbkdf2_hmac_sha512(pw, salt, iters, out)` | `src/pbkdf2.rs` | writes `[u8; 64]` |

## Status

| Primitive | Status |
|---|---|
| SHA-256 | IMPLEMENTED; TESTED transitively through the BIP39/BIP32 vectors |
| SHA-512 / HMAC-SHA512 | IMPLEMENTED; TESTED through the derivation vectors |
| PBKDF2-HMAC-SHA512, 2048 rounds | IMPLEMENTED; TESTED against BIP39 seed vectors |

These are not fuzzed in isolation; their correctness is established by the higher
layers whose published vectors they have to reproduce exactly. A wrong SHA-512
would fail every BIP32 vector immediately.

## Source

`userland/nonos_hd/src/{sha256,sha512,hmac512,pbkdf2}.rs`. Read `pbkdf2.rs` last;
it is the one that ties the primitives to BIP39 and shows the wiping discipline
the rest of the crate follows.

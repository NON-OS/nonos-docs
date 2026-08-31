# BIP32 and BIP44: the derivation tree

BIP32 turns one 64-byte seed into an unbounded tree of keys, each addressed by a
path like `m/44'/60'/0'/0/0`. BIP44 is the convention that gives the levels of
that path meaning (purpose, coin, account, change, index). `nonos_hd` implements
the tree in `userland/nonos_hd/src/bip32/` and the standard Ethereum path in
`src/path.rs`.

## The extended key

Every node in the tree is an extended private key, `Xprv`
(`userland/nonos_hd/src/bip32/xprv.rs`): a 32-byte private key and a 32-byte
chain code. The chain code is the extra entropy that makes derivation
deterministic but not reversible from the key alone; you need both halves to
derive a child, and knowing a child key does not give you a sibling.

## The master key

`master_from_seed` (`userland/nonos_hd/src/bip32/master.rs`) is the root. It is
`HMAC-SHA512("Bitcoin seed", seed)`: the left 32 bytes become the master private
key, the right 32 the master chain code. It returns `None` for a seed outside the
16-to-64-byte range, and for the astronomically unlikely case that the derived
key is not a valid secp256k1 scalar (zero, or at least the curve order), which
the standard says to reject rather than clamp. `is_valid_scalar`
(`src/bip32/scalar.rs`) is that check, and the intermediate HMAC output is wiped
before the function returns.

## Hardened and non-hardened children

This is the part that carries the whole security argument, so it is worth being
precise. A child index below `0x8000_0000` is non-hardened; at or above it, the
`HARDENED` offset, it is hardened.

**Hardened** (`child_hardened`, `userland/nonos_hd/src/bip32/child.rs`) computes

```
I = HMAC-SHA512(parent_chain, 0x00 || parent_private_key || ser32(index'))
```

where `index'` carries the hardened offset. It needs only the parent private key,
no public key, which is exactly why the account-level path elements of BIP44 are
all hardened: hardened derivation cannot be run by someone holding only a public
key, so it firewalls the account tree from a leaked extended public key.

**Non-hardened** (`child_normal`, same file) computes

```
I = HMAC-SHA512(parent_chain, ser_P(parent_public_key) || ser32(index))
```

It needs the compressed parent public key, and it requires the index to be below
the hardened offset. In both cases the left half of `I` is added, modulo the
curve order, to the parent private key to get the child private key, and the
right half is the child chain code.

The public key the non-hardened step needs does not come from this crate. It
comes from a provider the caller supplies. That seam is the subject of
[security.md](security.md); the short version is that a bespoke secp256k1 scalar
multiply is a classic place to leak a key through timing, so the crate refuses to
carry one and takes the point from kernel-proven or audited code.

`compress_pubkey` (`userland/nonos_hd/src/bip32/compress.rs`) turns the 65-byte
uncompressed SEC1 public key the provider returns into the 33-byte compressed
form the non-hardened derivation hashes.

## The Ethereum path

`derive_eth_key` (`userland/nonos_hd/src/path.rs`) walks the standard Ethereum
account path `m/44'/60'/0'/0/0` from a seed to an account private key. The first
three steps (`44'`, `60'`, `0'`) are hardened and need no public key; the last
two (`0`, `0`) are non-hardened and go through the provider. Its signature is:

```rust
pub fn derive_eth_key<F>(seed: &[u8; 64], mut pubkey: F, out: &mut [u8; 32]) -> bool
where F: FnMut(&[u8; 32]) -> Option<[u8; 65]>
```

The closure `pubkey` is the secp256k1 provider: given a 32-byte secret it returns
the 65-byte uncompressed public key, or `None`. The function wipes `out` first,
walks the five levels, wipes every intermediate extended key as it goes, and on
any failure returns `false` with a zeroed output. As with the seed derivation,
there is no partial result: you get the exact account key or nothing.

## Path convention (BIP44)

`m / 44' / 60' / 0' / 0 / 0` reads as purpose 44 (BIP44), coin type 60
(Ethereum), account 0, external chain, address index 0. Changing the last index
walks to the next address in the same account; changing the account index (a
hardened level) walks to an independent account. The crate hardcodes the Ethereum
coin type in `derive_eth_key`; other coins would be another path walk over the
same child functions.

## Status

| Piece | Source | Status |
|---|---|---|
| Extended key `Xprv` | `src/bip32/xprv.rs` | IMPLEMENTED |
| Master from seed (+ scalar validity) | `src/bip32/master.rs`, `scalar.rs` | IMPLEMENTED; TESTED against BIP32 vectors |
| Hardened child | `src/bip32/child.rs` | IMPLEMENTED; TESTED |
| Non-hardened child | `src/bip32/child.rs` | IMPLEMENTED; TESTED; needs the provider |
| Public-key compression | `src/bip32/compress.rs` | IMPLEMENTED |
| Ethereum path `m/44'/60'/0'/0/0` | `src/path.rs` | IMPLEMENTED; TESTED end to end |

## Source

`userland/nonos_hd/src/bip32/` and `src/path.rs`. Read `master.rs`, then
`child.rs` for the hardened and non-hardened split, then `path.rs` to see the two
kinds of step composed into a real account key. The provider seam is in
[security.md](security.md).

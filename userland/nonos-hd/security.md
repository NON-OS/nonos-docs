# Security: the trust boundary of nonos_hd

This crate produces private keys. That makes its trust boundary more important
than its features, so this page states exactly what it does and does not
guarantee, what it delegates and why, and what its correctness actually rests on.

## What it owns, and why that is safe to own

The hash and key-derivation math is the crate's own: SHA-256, SHA-512,
HMAC-SHA512, and PBKDF2-HMAC-SHA512 (see [primitives.md](primitives.md)). Writing
these by hand is defensible because they are well-trodden, they have no secret
branches or table lookups indexed by secret data in this implementation, and,
crucially, they are testable against published vectors. A wrong hash does not
hide; it fails every BIP39 and BIP32 vector immediately. The correctness of this
layer is therefore established, not asserted.

## What it delegates, and why that is the right call

Non-hardened BIP32 derivation needs a point multiplication on the secp256k1
curve. The crate deliberately does not implement one. Instead `child_normal` and
`derive_eth_key` take the parent public key from a caller-supplied provider (the
`pubkey` closure, `FnMut(&[u8; 32]) -> Option<[u8; 65]>`).

The reason is specifically about side channels. A constant-time secp256k1 scalar
multiply is hard to get right, and a version that is functionally correct but
leaks the scalar through timing or cache behavior looks exactly like a working
one right up until a private key walks out through the side channel. So the crate
refuses to be that code. In the keyring capsule the provider is the kernel's
proven `crypto_secp256k1_pubkey` syscall; in host tests it is the audited `k256`
crate. Either way the curve operation runs in code that is kernel-proven or
externally audited, and the crate is only as side-channel-safe on that one
operation as the provider it is handed. That is stated so a future caller does
not wire in a naive provider and quietly lose the property.

## Key material does not linger

`wipe` (`userland/nonos_hd/src/wipe.rs`) zeroizes key material after use, and the
crate applies it consistently:

- Every intermediate PBKDF2 `U` value is wiped as it is folded into the seed.
- The assembled mnemonic phrase buffer is wiped before `seed_from_words` returns.
- The master-key HMAC output is wiped after its halves are split out.
- Every intermediate extended key in a path walk is wiped as the walk proceeds,
  and on any failure the output key is zeroed.

On a RAM-resident, amnesic system the entire session is gone at power-off, so
this is not the last line of defense. It is the discipline that keeps a seed or
an extended private key from sitting in a stack buffer for longer than the
derivation that needs it, which shortens the window in which a bug elsewhere
could read it.

## What correctness rests on

The vectors. `tests/vectors.rs` and `tests/data/` run the crate against the
official BIP39 and BIP32 test vectors on the host. This is the same posture the
git crate takes against real `git`: the standard's own outputs are the oracle,
and the interop claim is measured, not argued. There is no separate formal proof
of the derivation math in the tree.

## Threats it does and does not address

- **Wrong input.** A mistyped or reordered mnemonic is caught by the BIP39
  checksum before any key is derived (see [bip39.md](bip39.md)). ADDRESSED.
- **Non-standard derivation.** A key that does not match the vectors fails the
  test suite. ADDRESSED at build time.
- **Side channels in the curve op.** Delegated to the provider; only as good as
  that provider. PARTIALLY ADDRESSED, by construction, not by this crate.
- **Side channels in the hash/KDF math.** The implementations avoid
  secret-dependent branching and indexing, but there is no formal constant-time
  proof. NOT PROVEN, believed sound.
- **Key handling after derivation.** Once a key leaves this crate it is the
  wallet's and keyring's problem. OUT OF SCOPE here; see the wallet custody docs.

## Status summary

| Property | Status |
|---|---|
| Derivation matches BIP39/BIP32 vectors | TESTED (the correctness contract) |
| Input validation (checksum) | ENFORCED |
| Key-material zeroization | IMPLEMENTED, applied throughout |
| secp256k1 point multiply | DELEGATED to a proven/audited provider |
| Constant-time hash/KDF | believed sound, NOT PROVEN |

## Source

`userland/nonos_hd/src/wipe.rs` for the zeroization, `src/path.rs` and
`src/bip32/child.rs` for the provider seam, and `tests/` for the vectors that
keep the whole crate honest.

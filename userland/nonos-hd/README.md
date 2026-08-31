# nonos_hd: BIP39 and BIP32 key derivation

The crate that turns a mnemonic into wallet keys. It implements BIP39 mnemonics
and BIP32/BIP44 hierarchical deterministic derivation for the keyring capsule,
`no_std`, allocation-free, and from scratch. It is small, security-critical, and
one of the few places where "from scratch" is a liability unless it is proven,
so it is proven against the official test vectors.

`userland/nonos_hd/`.

## The shape of it

The crate is a stack. Each layer is documented on its own page:

- **[primitives.md](primitives.md)** the hash and KDF math it owns: SHA-256,
  SHA-512, HMAC-SHA512, and PBKDF2-HMAC-SHA512 with the real 2048 rounds.
- **[bip39.md](bip39.md)** the mnemonic layer: entropy to words and back with
  the checksum that rejects a mistyped phrase, and the seed derivation.
- **[bip32.md](bip32.md)** the derivation tree: the extended key, the master
  key, hardened and non-hardened children, and the BIP44 Ethereum path.
- **[security.md](security.md)** the trust boundary: what is delegated and why,
  how key material is wiped, and what the correctness rests on.

## The one thing to understand first

Everything in this crate is deterministic and pure, and it is checked against the
one oracle that matters, real BIP39 and BIP32 test vectors (`tests/vectors.rs`,
`tests/data/`). If it derived a different key than the published vectors, a
wallet built on it would silently send funds to the wrong address, so the
vectors are the correctness contract, not a nicety.

The single primitive it does not implement is secp256k1 point multiplication.
Non-hardened derivation takes the parent public key from a caller-supplied
provider, which in the capsule is the kernel's proven `crypto_secp256k1_pubkey`
syscall and in host tests is the audited `k256` crate. This is deliberate and is
explained in [security.md](security.md): the well-trodden, vector-testable hash
math it owns; the side-channel-sensitive curve operation it delegates to code
that is kernel-proven or externally audited.

## Public surface

From `userland/nonos_hd/src/lib.rs`:

| Export | Page |
|---|---|
| `sha256`, `sha512`, `Sha512`, `hmac_sha512`, `HmacSha512`, `pbkdf2_hmac_sha512` | [primitives.md](primitives.md) |
| `ENGLISH_WORDLIST`, mnemonic conversion, `seed_from_words` | [bip39.md](bip39.md) |
| `derive_eth_key` and the BIP32 tree | [bip32.md](bip32.md) |
| `wipe` | [security.md](security.md) |

## Status

Fully summarised per layer on each page. In one line: IMPLEMENTED and TESTED
against the official vectors for everything except the secp256k1 point multiply,
which is DELEGATED.

## Source

`userland/nonos_hd/src/`. `lib.rs` for the surface, then follow the pages above
in order, which is also the order the code is layered.

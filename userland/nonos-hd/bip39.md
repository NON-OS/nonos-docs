# BIP39: mnemonics and the seed

BIP39 is the layer a person actually sees: the twelve to twenty four words you
write down. Its job is to encode wallet entropy as words that are hard to
transcribe wrong, catch it when they are transcribed wrong anyway, and stretch
the words into the 64-byte seed the derivation tree starts from. `nonos_hd`
implements all three parts in `userland/nonos_hd/src/bip39/` and the wordlist in
`src/wordlist.rs`.

## Entropy to words

`entropy_to_words` (`userland/nonos_hd/src/bip39/to_words.rs`) takes 16, 20, 24,
28, or 32 bytes of entropy (128 to 256 bits, the standard's five sizes) and
produces word indices. The encoding is exact BIP39: append the first
`entropy_bits / 32` bits of the entropy's SHA-256 as a checksum, then read the
combined bit string out in 11-bit groups, each group indexing one of the 2048
words. It writes indices into a caller buffer and returns the word count, or
`None` for an entropy length outside the standard. Twenty four words is 256 bits
of entropy plus an 8-bit checksum, which is `264 / 11 = 24` groups.

## Words back to entropy, with the checksum enforced

`words_to_entropy` (`userland/nonos_hd/src/bip39/from_words.rs`) is the reverse,
and it is where a transcription error dies. It rebuilds the entropy from the word
indices and recomputes the checksum from the entropy's SHA-256; if the appended
checksum bits do not match, it returns `None`. The consequence is stated plainly
in the source: a wrong word count, an out-of-range index, or a checksum mismatch,
which is what a mistyped or reordered phrase produces, is rejected here, before
any key is derived from it. That ordering is the point. The system never derives
a seed from something it has not first confirmed is a valid mnemonic, so a fat
finger fails loudly at entry instead of quietly producing a different, empty
wallet.

The word-to-index lookup itself is `word_index`
(`userland/nonos_hd/src/bip39/word_index.rs`) against `ENGLISH_WORDLIST`
(`src/wordlist.rs`), the standard 2048-word English list.

## Words to seed

`seed_from_words` (`userland/nonos_hd/src/bip39/seed.rs`) is the stretch. It
space-joins the words into a phrase, salts with the ASCII `"mnemonic"` followed
by the optional passphrase, and runs `pbkdf2_hmac_sha512` for 2048 rounds (see
[primitives.md](primitives.md)) to produce the 64-byte seed. Everything is done
in fixed buffers: `PHRASE_MAX` bounds the joined phrase at twenty four words of
at most eight letters plus separators, `PASSPHRASE_MAX` bounds the passphrase at
128 bytes, and the assembled phrase buffer is wiped before return.

It returns `false` with a zeroed output on an invalid index or an oversized
passphrase, and the guarantee that goes with that is worth quoting: it never
returns a seed derived from something other than the actual words. There is no
partial or best-effort seed. Either the inputs are valid and you get the exact
BIP39 seed, or you get nothing and a zeroed buffer.

## The BIP39 passphrase

The passphrase is the "25th word": an optional secret that is mixed into the salt
and so changes the entire derived seed. A wallet with the same words and a
different passphrase is a completely different wallet, which is both a feature
(plausible deniability, a hidden wallet) and a footgun (forget it and the funds
are gone, because there is no record of it anywhere). This crate implements it
faithfully and takes no position on how the wallet uses it; it just makes sure a
128-byte cap and a wipe bound the handling.

## Status

| Piece | Source | Status |
|---|---|---|
| Entropy to words (+ checksum) | `src/bip39/to_words.rs` | IMPLEMENTED; TESTED against BIP39 vectors |
| Words to entropy (checksum enforced) | `src/bip39/from_words.rs` | IMPLEMENTED; ENFORCED (rejects bad phrases) |
| Word index / English wordlist | `src/bip39/word_index.rs`, `src/wordlist.rs` | IMPLEMENTED |
| Seed derivation (PBKDF2, 2048) | `src/bip39/seed.rs` | IMPLEMENTED; TESTED against BIP39 seed vectors |

## Source

`userland/nonos_hd/src/bip39/`. Read `to_words.rs` and `from_words.rs` together
to see the encoding and its inverse, then `seed.rs` for the stretch into the
seed that [bip32.md](bip32.md) consumes.

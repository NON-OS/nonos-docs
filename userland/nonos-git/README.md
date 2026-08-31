# nonos_git: git from scratch, in the terminal

The NONOS terminal has `git`, and it is not a port of libgit2 or a shell-out to
a binary that does not exist on this system. It is a `no_std` reimplementation of
git's object model, hashing, object store, packfile format, and smart transport,
written so that a repository produced here is one real `git` can read, and a
repository real `git` produced is one this reads. `userland/nonos_git/` is that
implementation.

The reason it can be from scratch and still correct is that git's core is
small and exactly specified. A repository is a content-addressed store: every
blob, tree, and commit is named by the SHA-1 of its framed bytes,
`<type> <size>\0<content>`. Get the framing and the hash right and everything
else is bookkeeping on top of it. This crate owns that core and proves it, then
the store, the refs, and the commands build up from there.

## The layers, bottom to top

**Hash and object framing.** `sha1` (`userland/nonos_git/src/sha1/`) is the
hash, implemented here because there is no system one to lean on. `object`
(`src/object/`) does the framing: `frame` wraps content as
`<type> <size>\0<content>`, `unframe` reverses it, and `ObjectKind` is the blob,
tree, commit, tag distinction. `oid` (`src/oid/`) is `ObjectId`, the SHA-1 of the
framed bytes, which is the name every object is stored and referenced under. This
is the layer that has to agree with real git bit for bit, and it is pure and
deterministic, so it is tested on the host against the hashes real `git` produces
for the same input.

**Compression.** `zlib` (`src/zlib/`) is `compress` and `decompress` with a real
inflate, because git objects and packfiles are zlib-deflated and there is no
system zlib either. `InflateError` is its failure surface.

**The object database.** `odb` (`src/odb/`) is `read_object` and `write_object`:
given an `ObjectId`, find and inflate the loose object or the packed one; given
content, frame it, hash it, deflate it, and write it. `storage` (`src/storage/`)
is the `Storage` trait underneath, the actual bytes-on-disk boundary, so the
object database does not care whether it is talking to a real filesystem or a
capsule-backed store.

**Packfiles.** `pack` (`src/pack/`) reads and writes the packed format that git
uses for transport and for compact storage: `read_pack`, `write_pack`,
`PackObject`, and the delta resolution in `pack/delta` that reconstructs objects
stored as deltas against others. This is the part that makes fetch and clone
tractable, because a remote sends a pack, not ten thousand loose objects.

**Trees, commits, the index.** `tree` (`src/tree/`) parses and encodes tree
objects (`TreeEntry`, `Mode`), `commit` (`src/commit/`) does commits with their
`Signature` (author and committer, name, email, time), and `index` (`src/index/`)
is the staging index (`IndexEntry`) that sits between the working tree and the
next commit. These are the objects a user actually thinks in.

**Refs.** `refs` (`src/refs/`) is HEAD and branches: `read_head`,
`resolve_head`, `set_head_branch`, `update_ref`, and `is_valid_ref_name`. `Head`
is the current position, symbolic or detached. This is the mutable layer over the
immutable object store.

**Repository commands.** `repo` (`src/repo/`) ties it together into the verbs a
user runs: building a tree from the index (`repo/tree_build`), checking it out
(`repo/checkout`), and the clone and push flows (`repo/clone`, `repo/push`).

**Remote and wire.** `remote` (`src/remote/`) is `clone`, `discover`, `fetch`,
and `push`. `transport` (`src/transport/`) is the `Transport` trait it runs over,
and `wire` (`src/wire/`) is git's smart protocol on top of that: the pkt-line
framing (`wire/pkt`), the ref advertisement (`wire/advert`), and the update
request (`wire/update`). This is how the terminal's `git` talks to a real remote
over the network stack.

## Why it is trustworthy

Two things. The core is deterministic and pure, and it is checked against the one
oracle that matters: real git. If `nonos_git` hashes a blob to a different
`ObjectId` than `git hash-object` would, the test fails, so the interop claim is
not asserted, it is measured. And the layering means each piece can be wrong in
isolation and caught: a bad inflate shows up in `zlib` tests, a bad delta in
`pack`, a bad frame in `object`, without having to run a whole clone to find out.

## What to be careful about

- It is the object model and transport, not a full git CLI. The set of commands
  is what `repo` and the terminal expose, not everything `git` has.
- SHA-1 is what git uses and what this matches; it inherits git's ongoing SHA-1
  situation, no better and no worse.
- Correctness rests on the host tests against real git. There is no separate
  formal proof; the oracle is `git` itself.

## Status

| Layer | Source | Status |
|---|---|---|
| SHA-1 | `src/sha1/` | IMPLEMENTED; TESTED against real git hashes |
| Object framing and id | `src/object/`, `src/oid/` | IMPLEMENTED; TESTED (bit-for-bit vs git) |
| zlib compress/inflate | `src/zlib/` | IMPLEMENTED |
| Object database | `src/odb/`, `src/storage/` | IMPLEMENTED |
| Packfiles and deltas | `src/pack/` | IMPLEMENTED |
| Trees, commits, index | `src/tree/`, `src/commit/`, `src/index/` | IMPLEMENTED |
| Refs and HEAD | `src/refs/` | IMPLEMENTED |
| Repo commands (clone, checkout, push) | `src/repo/` | IMPLEMENTED |
| Remote and smart wire protocol | `src/remote/`, `src/transport/`, `src/wire/` | IMPLEMENTED |

The proven property is interoperability of the object core with real git. The
network paths are exercised through the terminal against real remotes.

## Source

`userland/nonos_git/src/`. Read `lib.rs` for the full public surface, then
`object/` and `oid/` for the framing and naming that everything else is built
on, `odb/` for the store, `pack/` for transport-sized objects, and `remote/` with
`wire/` for how it talks to a real server.

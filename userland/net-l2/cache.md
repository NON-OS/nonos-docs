# The ARP neighbour cache

The neighbour cache is where the capsule's only durable state lives, and it is bounded by construction. This
page mirrors `src/arp/cache/`, one file per concern: the cache struct and its constants, the learn policy,
lookup, insert, oldest-first eviction, the pending-request ring, and the constructor. For who calls into the
cache, the resolve op and the ingress observer, see the [framing](framing.md) and [operations](operations.md)
pages.

## The cache structure

`Cache` holds a fixed array of `ENTRY_CAP` optional entries, a live count, a monotonically increasing
sequence counter, a fixed `PENDING_CAP` ring of pending target IPs, and a head index into that ring
(`src/arp/cache/cache_type.rs:20`). The capacities are compile-time constants: 64 entries and 8 pending
slots (`src/arp/cache/constants.rs:17`). Nothing here grows: both arrays are fixed-size, so the cache cannot
be pushed past its bound no matter how much ARP traffic arrives. `Cache::new` is a `const fn` that zeroes the
arrays, the count, the sequence, and the head, which is what lets the whole cache live in a static behind a
mutex (`src/arp/cache/new.rs:21`, `src/state.rs:34`).

An `Entry` is an IPv4 address, a MAC, and the sequence number stamped when it was last written
(`src/arp/cache/entry.rs:20`). The sequence is the age key eviction uses.

## The learn policy

`decide` is the single point that says whether an observed sender binding may enter the cache, and it is
pure: it takes the existing MAC for that IP, the sender MAC from the packet, and whether the packet was
solicited, and returns one of `Insert`, `Refresh`, or `Reject` (`src/arp/cache/learn.rs:24`). The rules are:

- An existing binding whose MAC matches the sender is a `Refresh`, moving it to the front of the age order
  without changing the MAC.
- An existing binding whose MAC differs from the sender is a `Reject`. A packet cannot silently rebind an
  IP that is already mapped to a different MAC.
- No existing binding, and the packet was solicited, is an `Insert`.
- No existing binding, and the packet was unsolicited, is a `Reject`. A gratuitous ARP for an address the
  capsule never asked about plants nothing.

The caller in `on_inbound` applies the result: `Insert` and `Refresh` both call `insert`, and `Reject` does
nothing (`src/arp/handle.rs:42`). The solicited flag itself is computed one layer up, from the pending ring
and the interface IP (`src/arp/handle.rs:40`).

## Lookup

`lookup` scans the live entries for a matching IPv4 and returns its MAC, or `None`
(`src/arp/cache/lookup.rs:21`). It is a linear scan over the fixed array, which is bounded by the 64-entry
cap. `OP_ARP_RESOLVE` calls it first and answers directly from the cache on a hit
(`src/server/handlers/arp_resolve.rs:35`).

## Insert and eviction

`insert` stamps a fresh sequence number and advances the counter with a wrapping add
(`src/arp/cache/insert.rs:23`). If the IP is already present it updates the MAC and the sequence in place and
returns, so a refresh does not consume a new slot (`src/arp/cache/insert.rs:26`). Otherwise it takes the
first empty slot (`src/arp/cache/insert.rs:34`). Only if there is no empty slot does it evict: `evict_oldest`
scans for the entry with the smallest sequence number, the least recently written, clears it, and drops the
count, after which insert takes the freed slot (`src/arp/cache/insert.rs:39`, `src/arp/cache/evict_oldest.rs:20`).
This is the bounded-growth guarantee in practice: a full cache admits a new neighbour by dropping the oldest
one, never by allocating.

## The pending ring

The pending ring records which target IPs the capsule has an outstanding ARP request for, so a later reply
can be recognised as solicited. `note_pending` adds a target, skipping it if already present, and advances
the head with a modular wrap so the ring reuses its 8 slots (`src/arp/cache/pending.rs:21`). `is_pending`
looks a target up and, on a hit, clears the slot and returns true, so a pending request is consumed exactly
once when its reply arrives (`src/arp/cache/pending.rs:29`). `OP_ARP_RESOLVE` calls `note_pending` on a
cache miss before broadcasting its request (`src/server/handlers/arp_resolve.rs:46`), and `on_inbound` calls
`is_pending` to decide whether an inbound reply was solicited (`src/arp/handle.rs:40`).

## Source map

```
  userland/capsule_net_l2/src/arp/cache/cache_type.rs   the Cache struct fields
  userland/capsule_net_l2/src/arp/cache/constants.rs    ENTRY_CAP (64) and PENDING_CAP (8)
  userland/capsule_net_l2/src/arp/cache/entry.rs        the Entry struct with its sequence key
  userland/capsule_net_l2/src/arp/cache/new.rs          the const constructor
  userland/capsule_net_l2/src/arp/cache/learn.rs        decide: the Insert/Refresh/Reject policy
  userland/capsule_net_l2/src/arp/cache/lookup.rs       lookup by IPv4
  userland/capsule_net_l2/src/arp/cache/insert.rs       insert with in-place refresh and eviction fallback
  userland/capsule_net_l2/src/arp/cache/evict_oldest.rs oldest-sequence eviction
  userland/capsule_net_l2/src/arp/cache/pending.rs      note_pending and is_pending over the ring
  userland/capsule_net_l2/src/arp/handle.rs             the caller that applies the learn decision
  userland/capsule_net_l2/src/server/handlers/arp_resolve.rs  the resolve op that reads and marks the cache
```

Every reference above is verified against those trees.
</content>

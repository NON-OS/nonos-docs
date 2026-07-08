# capsule_entropy

`capsule_entropy` is the userland random-bytes service. It is deliberately thin: it draws bytes directly
from the CPU hardware random generator, serves them under a per-request cap, and keeps four counters so
the entropy path is observable. It holds no CSPRNG state of its own, it is a monitored pass-through to the
hardware source, which is why it is honest to call it a wrapper rather than a generator. Service
`entropy_pool` on port 4100, reply endpoint `0x1_0000_0003`, capability mask `0x39`. The source is
`userland/capsule_entropy/`.

## Contents

- [The server loop](#the-server-loop)
- [The wire protocol](#the-wire-protocol)
- [The RDRAND fill](#the-rdrand-fill)
- [GET_RANDOM](#get_random)
- [RESEED and the kernel-owned RNG](#reseed-and-the-kernel-owned-rng)
- [State: the observability counters](#state-the-observability-counters)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The server loop

`main.rs:26` initializes the heap and calls `server::run` (`src/server/runner.rs:27`), which runs the
request loop over a 4608-byte buffer with a magic-and-version framed decode:

```
  run():
      pool = Pool::new()                       // four AtomicU64 counters, all zero
      loop:
          n = mk_ipc_recv(inbox 0, buf)
          req  = decode_request(buf[..n])       // magic 0x4E4F454E "NOEN", version 1; bounds
                 on decode error -> reply EINVAL immediately (do not drop)
          resp = dispatch(pool, req)
          mk_ipc_send(KERNEL_REPLY_ENDPOINT, resp)
```

A decode error is answered with `EINVAL` rather than silently dropped, so a malformed request does not
leave the caller waiting for a reply that never comes.

## The wire protocol

The frame is a 20-byte header, `u32 magic (0x4E4F454E "NOEN"), u16 version, u16 op, u16 flags, u16
reserved, u32 request_id, u32 payload_len`, and the decode validates the magic, the version, and that the
declared payload length fits the frame before dispatch. Four operations:

```
  1  GET_RANDOM    2  GET_STATS    3  RESEED    4  HEALTHCHECK
```

The bounds are `MAX_RANDOM_BYTES = 4096` per draw and `MAX_RESEED_BYTES = 256` per reseed. The 4 KiB cap
on a single draw is deliberate: it stops a caller from forcing a large allocation through one request.

## The RDRAND fill

The heart of the capsule is `rdrand_fill` (`src/pool/fill.rs:23`), which draws from the x86 hardware
random generator word by word:

```
  #[target_feature(enable = "rdrand")]
  rdrand_fill(out):
      filled = 0
      while filled < out.len():
          word = 0; tries = 0
          while _rdrand64_step(&word) != 1:        // RDRAND can transiently fail
              tries += 1
              if tries >= 32:  return false          // give up on a stalled source
          take = min(8, out.len() - filled)
          out[filled..filled+take] = word.to_le_bytes()[..take]
          filled += take
      return true
```

Each 64-bit `RDRAND` is retried up to 32 times, because the instruction is allowed to fail transiently
when its entropy buffer is momentarily empty; after 32 failures the fill gives up and reports a
source failure. The output is filled eight bytes at a time, with the last chunk truncated to the
requested length. There is no software mixing, the bytes are the hardware generator's output verbatim.

## GET_RANDOM

`get_random` (`src/server/handlers/getrandom.rs:27`) parses a length, caps it, fills, and checks the
result exactly:

```
  get_random(pool, req):
      if payload < 4:                 EINVAL
      length = u32(payload[0..4])
      if length > MAX_RANDOM_BYTES:   EMSGSIZE          // 4096 ceiling
      out = vec![0; length]
      n = pool.fill(out)              // caps to MAX_RANDOM_BYTES, updates counters
      if n < 0 or n != length:        EIO               // hardware source failure
      return out
```

`pool.fill` (`src/pool/fill.rs:42`) increments the `requests` counter before the attempt, returns `-5` and
increments `source_failures` if `rdrand_fill` gave up, and otherwise adds `want` to `bytes_served`. So a
successful request returns exactly the requested number of hardware-random bytes, and a hardware failure
is a clean `EIO`.

## RESEED and the kernel-owned RNG

`reseed` (`src/server/handlers/reseed.rs`) bounds-checks the supplied entropy (up to
`MAX_RESEED_BYTES = 256`), increments the reseed counter for observability, and acknowledges. It does
**not** mix the supplied bytes into any state, because there is no userland CSPRNG state to mix into: the
random source is kernel-owned (the [kernel secure RNG](../../subsystems/crypto/randomness.md)), and the
cap enforcement and any actual reseeding happen on the kernel side before a call reaches this capsule. So
`RESEED` here is an observability breadcrumb, and this page states that rather than implying the capsule
reseeds a generator it owns.

## State: the observability counters

The `Pool` (`src/pool/types.rs:26`) is four relaxed atomics and nothing else:

```
  requests             total GET_RANDOM calls
  bytes_served         cumulative bytes returned
  last_reseed_request  the last reseed timestamp
  source_failures      RDRAND give-ups
```

`GET_STATS` returns a `Stats` snapshot of these, so an operator can see how much randomness the system
has drawn and whether the hardware source has ever failed. There is no per-caller state; the source is
global.

## Security analysis

- **The source is the CPU hardware RNG** (`RDRAND`), retried and checked, with a clean `EIO` on failure
  rather than a silent fallback to a weak source.
- **The per-draw cap** (4 KiB) bounds the allocation a single request can force.
- **No per-caller state** and no secrets held, so there is nothing to leak between callers; the counters
  are non-sensitive.
- **Fail-closed decode**: a malformed frame is `EINVAL`, not a dropped request.

## Honest gaps

Stated plainly: there is no boot-time health check of `RDRAND` availability, so a persistent hardware
failure surfaces as `EIO` at request time rather than at startup; the capsule has no secondary RNG
fallback (by design, it trusts the kernel-routed hardware source); `GET_STATS` is open to any caller;
and the counters are not persisted across a restart. The reseed op does not itself reseed, as above.

## Source map

```
  userland/capsule_entropy/src/server/runner.rs      the loop, decode validation
  userland/capsule_entropy/src/server/handlers/getrandom.rs, get_stats.rs, reseed.rs, healthcheck.rs
  userland/capsule_entropy/src/pool/fill.rs          rdrand_fill + Pool::fill
  userland/capsule_entropy/src/pool/types.rs         Pool, Stats (the four counters)
  userland/capsule_entropy/src/protocol/             the NOEN frame, ops, caps
```

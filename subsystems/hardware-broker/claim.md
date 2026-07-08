# Device Claim and Epochs

Before a capsule can touch a device, it must claim it, and a device can be claimed by only
one capsule at a time. The claim is the root authority every later grant is checked against:
an MMIO mapping, a DMA buffer, an IRQ binding, or a port grant is only issued to the pid that
holds the claim, and only while the claim's epoch is still current. This page documents the
claim table and the epoch. The code is `src/hardware/broker/claim.rs`.

## The claim

A `Claim` (`src/hardware/broker/claim.rs:23`) binds a device to a holder and stamps it with an
epoch:

```
  struct Claim { pid: u32, device_id: u64, epoch: u64 }
```

The claims live in one global `Mutex<Vec<Claim>>`, and `claim` (`claim.rs:48`) refuses a
device that is already claimed:

```
  claim(pid, device_id):
      if any claim has this device_id:  AlreadyClaimed
      epoch = next_epoch()
      push Claim { pid, device_id, epoch }
      return epoch
```

Exclusivity is the first property: `AlreadyClaimed` means one capsule cannot claim a device
another already holds, so two drivers can never both be issued grants for the same hardware.
The claim errors, `UnknownDevice`, `AlreadyClaimed`, `NotHolder`, `NotClaimed`, are the four
distinct ways a claim operation can be refused.

## The epoch

The epoch is a monotonic counter (`claim.rs:39`) bumped on every successful claim. Its purpose
is to invalidate stale authority across a release-and-reclaim cycle. When a capsule claims a
device it receives the epoch; every grant request it later makes carries that `claim_epoch`,
and every grant path re-checks it:

```
  claim = lookup(device_id)          else NotClaimed
  if claim.pid != pid:               NotClaimed
  if claim.epoch != req.claim_epoch: StaleEpoch
```

If the device is released and claimed again, by the same capsule or a different one, the new
claim gets a fresh epoch, so any grant request still quoting the old epoch is rejected with
`StaleEpoch`. This closes the window where a capsule (or a bug) holds a grant handle from a
prior ownership and tries to use it after the device has changed hands. The epoch check appears
verbatim at the head of the [MMIO](mmio.md), [DMA](dma.md), and [IRQ](irq.md) paths.

## Release

`release` (`claim.rs:60`) drops a claim, but only for the holder: a `pid` that is not the
recorded holder gets `NotHolder`, and a device that is not claimed gets `NotClaimed`. Voluntary
release is one path; the other is `release_all_for_pid` (`claim.rs:74`), which retains only the
claims not held by a given pid and is called from the process exit path so a dying capsule
cannot leak a device claim. The [revocation](revocation.md) page covers how the claim drop is
coordinated with dropping the grants that depended on it.

## Source

```
  src/hardware/broker/claim.rs   Claim, the epoch counter, claim / release / release_all_for_pid
```

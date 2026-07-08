# The Message Envelope and Integrity

A routed message is carried in one structure, `IpcMessage`, and each one carries a keyed MAC
computed over its own fields. This page documents the envelope, the boot-time key that keys
the MAC, and the integrity check. The code is under `src/ipc/nonos_channel/`, whose live
surface is exactly the envelope (`message`), the MAC (`hash`), and the error type.

## The envelope

`IpcMessage` (`src/ipc/nonos_channel/message.rs:26`) is the on-the-wire form every routed
message takes:

```
  struct IpcMessage {
      from:         String,     // "proc.<pid>", stamped by the kernel
      to:           String,     // destination inbox name
      data:         Vec<u8>,
      timestamp_ms: u64,
      correlation:  u64,        // request/reply matching
      checksum64:   u64,        // keyed MAC, private field
  }
```

`new` (`message.rs:36`) rejects a payload larger than `MAX_MESSAGE_SIZE = 1 MiB`, stamps
the current millisecond timestamp, and computes the MAC before the message exists; there is
no way to construct an `IpcMessage` without a valid MAC over its contents, because
`checksum64` is private and set only in the constructors. `from` and `to` are set by the
routing layer, not by the sending capsule, so the sender identity in the envelope is the
kernel's attestation of who sent it.

## The MAC key

Integrity is keyed, not a bare hash, so it is seeded once at boot. `init_ipc_secret`
(`src/ipc/nonos_channel/hash.rs:25`) draws 32 bytes from the secure RNG, derives a key from
them with a domain-separated BLAKE3 derive-key, and stores it in a `Once`:

```
  init_ipc_secret():
      bytes = get_bytes_secure(32)               else "rng failed to seed IPC MAC key"
      key   = blake3::new_derive_key("NONOS:IPC:SECRET:v1").update(bytes).finalize()
      IPC_SECRET.call_once(|| key)
```

This runs during kernel init (`src/kernel_core/init/entry.rs:36`) and is **fatal on
failure**: if the RNG cannot seed the MAC key, the kernel does not continue with an
unkeyed IPC path. Every later MAC pulls the key through `get_ipc_secret`, which errors if
the key was never initialized, so a MAC can never be computed against a zero or absent key.

## The MAC

`compute_checksum` (`hash.rs:63`) is a keyed BLAKE3 MAC over the message's own fields, with
a domain-separation tag and a field separator so distinct messages cannot collide by
concatenation:

```
  mac = blake3::new_keyed(secret)
          .update("NONOS:IPC:MAC:v1")
          .update(from) .update(0xF0) .update(to)
          .update(timestamp_ms.le_bytes())
          .update(data)
          .finalize()
  checksum64 = last 8 bytes of mac, little-endian
```

The MAC binds the sender, the destination, the timestamp, and the payload together under
the boot secret. `validate_integrity` (`message.rs:72`) recomputes the MAC over the
current fields and compares; the comparison folds the difference into a single word and
tests it against zero in one step rather than short-circuiting byte by byte
(`hash.rs:83`), so the check does not leak where a forged MAC first diverges.

## Honest scope

The MAC is a sixty-four-bit tag, a deliberate size for an in-kernel integrity and
anti-corruption check rather than a full-width authentication tag against an adversary with
online forgery attempts; its job is to detect a corrupted or malformed envelope, and it is
keyed under the per-boot secret so the tag cannot be precomputed across boots. The envelope
travels inside the kernel between kernel-managed inboxes, so the transport itself is not an
untrusted channel; the MAC is defense in depth on the message structure, and the sender
identity guarantee comes from the kernel stamping `from`, not from the MAC alone.

## Source

```
  src/ipc/nonos_channel/message.rs   IpcMessage, MAX_MESSAGE_SIZE, validate_integrity
  src/ipc/nonos_channel/hash.rs      init_ipc_secret, compute_checksum, verify_checksum
  src/ipc/nonos_channel/error.rs     ChannelError
  src/kernel_core/init/entry.rs      the fatal boot-time seed of the MAC key
```

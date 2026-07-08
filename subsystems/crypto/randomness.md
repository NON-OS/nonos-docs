# Randomness

Every secret the kernel generates, the IPC MAC key, the capability key material, the kernel's
signing keypair, the ASLR offset, is drawn from one secure random path. That path combines a
software CSPRNG with hardware entropy so it does not depend on any single source. This page
documents the pipeline and what draws from it. The code is under `src/crypto/random_api/` and
`src/crypto/util/rng/`.

## The secure path

`get_bytes_secure` (`src/crypto/random_api/basic.rs:38`) is the two-stage fill:

```
  get_bytes_secure(buffer):
      rng::fill_random_bytes(buffer)     // software CSPRNG output
      mix_hardware_entropy(buffer)       // XOR hardware entropy over it
```

The software stage fills the buffer from the kernel CSPRNG. The hardware stage
(`hardware_mix.rs:20`) then mixes real hardware entropy into it by XOR, from two sources when they
are present:

```
  mix_hardware_entropy(buffer):
      mix_virtio_rng(buffer)   // if the virtio-rng driver is available, XOR its bytes in
      mix_cpu_rng(buffer)      // if RDRAND/RDSEED is available, XOR CPU entropy in
```

XOR-combining is the reason the result is no weaker than its best source: an output byte is the
software CSPRNG byte XOR the virtio-rng byte XOR the CPU-entropy byte, so an adversary would have
to predict all of the available sources to predict the output. The CPU source prefers `RDSEED`
over `RDRAND` (`next_cpu_random`), and the intermediate hardware buffers are zeroized after use so
they do not linger on the stack. When a hardware source is not present its stage is skipped, and
the software CSPRNG still fills the buffer.

## Entropy sizing

`get_bytes_checked` (`basic.rs:20`) refuses to fill a buffer smaller than the requested minimum
entropy (`required_entropy_bytes`, `entropy_check.rs`), returning `BufferTooSmall` rather than
handing back fewer bits than the caller asked for. This keeps a caller that needs, say, 256 bits
of key material from silently getting less.

## What draws from it

The secure path is the single source for the kernel's secrets:

```
  IPC MAC key         ipc/nonos_channel/hash.rs   init_ipc_secret, fatal if the RNG fails
  kernel keypair      crypto/kernel_keys.rs        the Ed25519 signing key at init
  capability keys     capabilities/token/          the MAC key material
  ASLR offset         elf/aslr/                     the per-load base randomization
```

The IPC MAC seeding is fatal on RNG failure (see [the IPC envelope](../ipc/envelope.md)): the
kernel will not run with an unkeyed IPC path. The others draw the same way, so the quality of every
kernel secret reduces to the quality of this one path, which is why it mixes hardware entropy
rather than trusting the software CSPRNG alone.

## Source

```
  src/crypto/random_api/basic.rs          get_bytes_secure, get_bytes_checked
  src/crypto/random_api/hardware_mix.rs    the virtio-rng and CPU-entropy XOR mix
  src/crypto/random_api/entropy_check.rs   the minimum-entropy sizing
  src/crypto/util/rng/entropy/hardware.rs  RDRAND / RDSEED access
```

# Consumers and Entropy

Time is a shared dependency: the scheduler sleeps on it, IPC stamps messages with it, the spawn
path ages tokens by it, and the RNG folds the raw counter into its entropy. This page collects who
reads the clock and how the timestamp counter doubles as an entropy source.

## Who reads the clock

The monotonic and wall-clock bases feed distinct consumers:

```
  scheduler       sleep_until deadlines and preemption timing   (monotonic now)
  IPC             IpcMessage.timestamp_ms + the MAC binds it      (wall-clock ms)
  capsule spawn   now_ms passed to preflight for cert/token validity windows
  page allocator  get_timestamp stamps each AllocatedPage         (raw TSC)
  boot log / shell  local time-of-day display
```

The split matters: the [scheduler](../scheduler/sleep-wake.md) uses monotonic time so a sleep is
immune to any wall-clock adjustment, while the [IPC envelope](../ipc/envelope.md) uses a wall-clock
millisecond timestamp because a message's time is a human-meaningful field, and it is bound into
the message MAC so it cannot be altered after the fact. The [spawn preflight](../elf-loader/integration.md)
takes a millisecond `now` so certificate and token validity windows are evaluated against real
time. Each reads through the `crate::time` façade rather than calling `rdtsc` directly.

## The counter as entropy

The raw timestamp counter is also an entropy input. The RNG folds `rdtsc` into its output on
several paths (`src/crypto/random_api/platform.rs:22`, `src/crypto/util/rng/global/generate.rs`):
the low bits of the counter at the moment of a draw carry timing jitter that an outside observer
cannot predict, so XOR-ing them into the generator adds unpredictability that does not depend on a
hardware RNG being present. This is a supplementary source, not the primary one; the primary secure
path and its hardware mixing are documented on the [randomness](../crypto/randomness.md) page. The
counter is a good jitter source precisely because it is high-resolution and free-running: two draws
a few instructions apart differ in their low bits by an amount that depends on cache, contention,
and interrupts.

## Source

```
  src/crypto/random_api/platform.rs        rdtsc folded into the RNG
  src/crypto/util/rng/global/generate.rs   rdtsc jitter in generation
  src/ipc/nonos_channel/message.rs         the wall-clock message timestamp
  src/scheduler/ (sleep)                    monotonic deadlines
```

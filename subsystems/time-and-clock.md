# Time and Clock

NØNOS keeps two notions of time: a monotonic counter for elapsed time and
scheduling, and a wall clock anchored to the real-time clock. Both are built on
the TSC, calibrated at boot. This page covers calibration, the two time bases,
the syscalls that expose them, and the entropy the system seeds at the same
stage. The [scheduler](scheduler.md) consumes the 100 Hz tick that lives
alongside all of this.

---

## Calibration at boot

Two early steps in core init set up time
(`src/boot/main/core_init.rs:24`):

```
  init_boot_time()    record RDTSC at boot in BOOT_TIME
  tsc::init_default() read the RTC for the boot wall-clock,
                      calibrate the TSC frequency
```

The TSC frequency is determined by CPUID where the CPU reports it, and by a
PIT-based calibration where it does not (`src/sys/timer/tsc.rs:62`). The result,
`TSC_FREQ_HZ`, plus the boot epoch read from the RTC, `BOOT_EPOCH_MS`, are the
two anchors every later time query is built from.

## The two time bases

```
  monotonic   elapsed since boot, from the TSC alone
                uptime_ms = (now_tsc - boot_tsc) * 1000 / TSC_FREQ_HZ

  wall clock   unix milliseconds, RTC boot time plus elapsed TSC
                unix_ms = BOOT_UNIX_MS + (now_tsc - boot_tsc) * 1000 / TSC_HZ
```

The monotonic base (`src/sys/timer/uptime.rs`) only ever increases and is what
scheduling and timeouts reason about. The wall clock (`src/sys/clock/core.rs`)
adds the RTC boot epoch so it reads as a real date and time. Both derive from the
same TSC delta, so they advance together; the wall clock is just offset to the
unix epoch.

## The time syscalls

```
  MkTimeMillis    unix-epoch milliseconds, as an i64
  MkTimeRtc       broken-down wall-clock time from the RTC
```

`MkTimeMillis` (`src/syscall/microkernel/time.rs:56`) returns
`sys::clock::unix_ms()` clamped to `i64::MAX`, and an error if the clock is not
yet ready. It returns an `i64`, not a `u64`: a client that stores it in an
unsigned type and then does a wrapping subtraction will compute nonsense, which
is the kind of bug that makes an on-screen clock stop updating.

`MkTimeRtc` (`src/syscall/microkernel/time.rs:33`) reads the RTC directly through
the CMOS ports (address on port 0x70, data on port 0x71,
`src/arch/x86_64/time/rtc/cmos.rs`) and writes a broken-down `year, month, day,
hour, minute, second` struct to a caller-supplied pointer. This is the calendar
time; `MkTimeMillis` is the millisecond timestamp.

On aarch64 and riscv64 the same two bases are built from those platforms'
monotonic counters (the generic timer, `mtime`) rather than the TSC, behind the
`read_time_counter` primitive on the [arch boundary](../architecture/overview.md).

## The preemption tick versus the TSC

The 100 Hz preemption timer and the TSC are different clocks doing different
jobs. The LAPIC timer fires an interrupt every 10 milliseconds and drives
scheduling: each tick decrements the running capsule's slice
(`src/arch/x86_64/interrupt/apic/preemption/install.rs:25`, `TICK_HZ = 100`). The
TSC is a free-running cycle counter used to calibrate frequency and to compute
elapsed and wall-clock time. The scheduler counts ticks; time queries read the
TSC. They are not derived from each other.

On a multiprocessor machine each CPU installs its own LAPIC timer at the same 100
Hz, so every core is preempted on its own cadence ([smp](smp.md)).

---

## Entropy, seeded at the same stage

Boot also seeds the system's randomness, because later capability tokens and
crypto depend on it. Two steps run in core init: `init_entropy` verifies the
hardware sources, and `init_rng` seeds the CSPRNG.

```
  sources, in preference order      src/crypto/util/entropy, src/crypto/util/rng
    VirtIO-RNG, if present
    RDSEED   (CPUID 7:0 EBX bit 18)
    RDRAND   (CPUID 1 ECX bit 30)
    TSC and timestamp jitter, mixed in through SHA-256
```

`init_rng` (`src/crypto/util/rng/global/init.rs`) collects a 32-byte seed from
those sources, initialises a ChaCha20-based CSPRNG, stores it in the global RNG,
and securely erases the seed. Capsules reach it through the `CryptoRandom`
syscall, which requires the `Crypto` capability and is routed to the entropy
service. The boot session nonce and the token signing key, both established in
this same boot phase, draw on this entropy, which is why every capability token
is bound to a value that is fresh per boot
([capabilities and tokens](../security/capabilities-and-tokens.md)).

There is also a lightweight, non-cryptographic per-CPU generator (an xorshift
seeded from the TSC, `src/smp/percpu`) used for things like layout randomisation.
It is deliberately separate from the CSPRNG and is never used where cryptographic
strength is required.

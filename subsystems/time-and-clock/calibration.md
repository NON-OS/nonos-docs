# TSC and Calibration

The kernel's clock is the CPU timestamp counter. `RDTSC` returns a monotonic tick count, and to
turn ticks into nanoseconds the kernel needs the counter's frequency, which it establishes once
at boot by calibration. This page documents the counter read and the calibration chain. The code
is under `src/arch/x86_64/time/`.

## Reading the counter

`rdtsc` reads the 64-bit timestamp counter (`src/arch/x86_64/time/timer/tsc.rs`), splicing the
`EDX:EAX` halves the instruction returns:

```
  rdtsc():
      asm "rdtsc" -> (hi: EDX, lo: EAX)
      (hi << 32) | lo
```

The counter increments at a fixed rate independent of the CPU's power state on modern parts, so
its difference over an interval is proportional to elapsed wall time; the constant of
proportionality is the frequency the calibration finds.

## The calibration chain

`calibrate` (`src/arch/x86_64/time/tsc/calibration/calibrate.rs:27`) establishes the frequency in
a fixed order of preference, recording the source and a confidence:

```
  calibrate():
      if not tsc_available:  NotAvailable
      boot_tsc = rdtsc()
      if get_cpuid_frequency() is Some(freq):
          source = Cpuid, confidence = 100     // authoritative, one sample
          return
      if calibrate_with_pit() is Ok((freq, confidence)):
          source = Pit                          // measured against the 8254 PIT
          return
      CalibrationFailed
```

The CPUID TSC-frequency leaf is tried first because it is authoritative: when the CPU reports its
own timestamp frequency, that value is exact and gets confidence 100. When CPUID does not report
it, the kernel falls back to measuring the counter against the 8254 PIT over a known interval,
which yields a frequency and a measured confidence over several samples. A HPET-based calibration
variant exists (`calibrate_with_hpet_base`) for platforms that expose one. If no method works,
calibration fails rather than guessing. The chosen source is recorded as a `CalibrationSource`
(`Cpuid`, `Pit`, ...), so the provenance of the frequency is observable.

## The stored frequency and the fallback

Calibration stores the frequency, the boot TSC, and the source. The nanosecond conversion in
`now_ns` (`src/arch/x86_64/time/timer/time.rs:22`) reads that stored frequency, and if it is still
zero (calibration has not run or failed) it substitutes a default of 2.5 GHz:

```
  now_ns():
      tsc_freq = TSC_FREQUENCY, or 2_500_000_000 if zero
      (( rdtsc() - boot_tsc ) * 1e9) / tsc_freq
```

The default keeps time monotonic and roughly sane on an uncalibrated boot rather than dividing by
zero or returning nothing, but a calibrated boot uses the real frequency. The subtraction is
saturating so the counter difference never underflows, which is what makes the derived time
monotonic.

## Source

```
  src/arch/x86_64/time/timer/tsc.rs                 rdtsc
  src/arch/x86_64/time/tsc/calibration/calibrate.rs  the CPUID -> PIT -> HPET chain
  src/arch/x86_64/time/tsc/calibration/cpuid.rs      the CPUID TSC-frequency leaf
  src/arch/x86_64/time/tsc/calibration/pit.rs        the PIT measurement fallback
  src/arch/x86_64/time/timer/time.rs                 now_ns and the frequency fallback
```

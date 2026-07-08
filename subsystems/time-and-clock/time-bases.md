# The Two Time Bases

The kernel keeps two notions of time from the one counter: a monotonic uptime that only ever
increases, used for scheduling deadlines and durations, and a wall-clock Unix time, used where a
human-meaningful timestamp is needed. Both derive from the calibrated TSC; they differ only in
their zero point. This page documents them and the façade that exposes them. The code is under
`src/arch/x86_64/time/`, re-exported as `crate::time` (`src/lib.rs:80`).

## Monotonic uptime

Monotonic time is elapsed time since boot, computed from the TSC delta (`timer/time.rs:22`):

```
  now_ns()  = (rdtsc() - boot_tsc) * 1e9 / tsc_freq
  now_ms()  = now_ns() / 1_000_000
```

Because the subtraction from the boot TSC saturates and the counter only counts up, `now_ns` never
goes backward. This is the base the [scheduler](../scheduler/sleep-wake.md) uses for sleep
deadlines and the durations any subsystem measures: it is unaffected by any wall-clock adjustment,
so a sleep or a timeout cannot be cut short or extended by the clock being set. `now_ns_checked`
returns `None` before the timer is initialized, so a caller can tell calibrated time from the
pre-init window.

## Wall-clock Unix time

Wall-clock time is the monotonic uptime added to a boot epoch (`timer/uptime.rs:38` and the clock
core):

```
  unix_timestamp_ms() = boot_unix_ms + uptime_ms()
```

The boot epoch is the real Unix time at boot, established once at init. `clock::init`
(`src/kernel_core/init/entry.rs:43`) seeds it from the bootloader handoff, which carries the TSC
frequency and the Unix epoch in milliseconds:

```
  clock::init(handoff.timing.fixed_freq_hz, handoff.timing.unix_epoch_ms)
```

When no boot epoch is available, the code reads the real-time clock hardware directly
(`arch::x86_64::time::rtc::read_unix_timestamp`) and adds the uptime, so a wall-clock timestamp is
always anchored to real time from either the handoff or the RTC. The local time-of-day helper
(`src/sys/clock/time.rs`) turns this Unix time into hours, minutes, and seconds, applying the
timezone offset from policy, which is what the boot log and the shell clock display.

## The façade

Callers throughout the kernel use `crate::time`, which is `arch::x86_64::time` re-exported: the
common entry points are `timestamp_millis` (wall-clock milliseconds) and the monotonic `now_ns` /
`now_ms`. Keeping the façade in the arch module is deliberate, the time source is inherently
architecture-specific (the counter and its calibration differ per ISA), so the shared kernel calls
`crate::time` and the arch layer provides the counter underneath, consistent with the
[multi-architecture](../smp/README.md) boundary the rest of the kernel follows.

## Source

```
  src/arch/x86_64/time/timer/time.rs     now_ns, now_ms (monotonic)
  src/arch/x86_64/time/timer/uptime.rs   unix_timestamp_ms (wall clock)
  src/arch/x86_64/time/rtc/              the real-time-clock read
  src/sys/clock/time.rs                  local time-of-day with timezone offset
  src/kernel_core/init/entry.rs          clock::init from the boot handoff
  src/lib.rs:80                          crate::time = arch::x86_64::time
```

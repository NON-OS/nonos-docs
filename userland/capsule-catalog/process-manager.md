# capsule_process_manager

`capsule_process_manager` is a GUI app that shows the running services and their CPU usage. Unlike the
service capsules, it has no IPC server: it is an [app-skeleton](../writing-an-app.md) application that
polls the kernel on a timer and paints a table with a per-service CPU sparkline. App
`app.process_manager` on port 4730, capability mask `0x1819`. The source is
`userland/capsule_process_manager/`.

## Contents

- [The app](#the-app)
- [State](#state)
- [Service discovery](#service-discovery)
- [CPU sampling](#cpu-sampling)
- [Rendering](#rendering)
- [Security analysis](#security-analysis)
- [Honest gaps](#honest-gaps)
- [Source map](#source-map)

## The app

`main.rs:27` hands `ProcessManager::new` to the skeleton's `run`. `ProcessManager`
(`src/pm/app.rs:25`) implements the `App` trait: a `manifest` for its window, an `on_event` that
refreshes on a key or click and closes on Escape, a `paint` that draws the table, and an `on_tick` that
samples the kernel every few ticks:

```
  on_tick(self):
      if self.ticks % 5 == 0:  self.state.refresh()   // re-resolve service pids
      self.ticks += 1
      sample(self.state)                               // read mk_proc_stat, update CPU history
      return true                                       // request a repaint
```

## State

The state (`src/pm/state.rs`) is a fixed set of eight monitored services (the terminal, file manager,
editor, settings, shell, and so on), each a row with a label, a service name, a pid, an online flag, and a
30-sample circular history of CPU percentage, plus the last total-tick and per-row tick snapshots for the
CPU delta computation.

## Service discovery

On refresh (`src/pm/state.rs`), the manager resolves each of the eight service names to a pid through the
app skeleton's `lookup_service`, marking the service online or offline and recording its pid. A service
that has crashed and not yet respawned shows offline until the next refresh (a few ticks later).

## CPU sampling

The interesting mechanism is the CPU-percentage computation (`src/pm/sample.rs:25`), which reads the
kernel's process-statistics syscall and computes each service's share of the total tick delta since the
last sample:

```
  sample(state):
      buf = mk_proc_stat(buf, MAX_ENTRIES = 64)        // header{total_ticks} + up to 64 {pid, run_ticks}
      dt = header.total_ticks - state.last_total_ticks
      warmed = state.last_total_ticks != 0 and dt > 0
      for each monitored row with pid != 0:
          ticks = find_ticks(buf, row.pid)
          if warmed:
              d = ticks - state.last_ticks[row]
              pct = min(d * 100 / dt, 100)             // this service's share of the interval
              row.cpu.percent[head] = pct; head = (head + 1) % HISTORY
          state.last_ticks[row] = ticks
      state.last_total_ticks = header.total_ticks
```

`mk_proc_stat` returns a header with the total system tick count and up to 64 per-process entries with a
pid and a run-tick count; the manager finds each monitored pid's ticks (`find_ticks`), computes the delta
since the last sample, and expresses it as a percentage of the total delta, clamped to 100. The first
sample is not "warmed" (there is no prior baseline), so a percentage is only computed from the second
sample on. The result feeds a circular history that the sparkline renders.

## Rendering

`paint` (`src/pm/paint.rs`) renders a table: a title, column headers (name, pid, caps), and a row per
monitored service with its label, its pid (or "offline"), and a small sparkline of the CPU history.

## Security analysis

The manager is a **read-only observer**: it reads `mk_proc_stat` and the service registry and displays
what they expose, and it does not start, stop, or signal any process. It holds no secrets and makes no
privileged calls beyond the introspection syscall.

## Honest gaps

Two limits are stated in the code itself: the capabilities column is a hardcoded "unavailable" because
per-process capability reporting is not implemented, and the CPU sampling is a rolling window (a sample
every few ticks, a roughly two-and-a-half-second history), so the sparkline is a recent trend rather than
an instantaneous reading, and a service pid can lag a respawn by one refresh interval. The introspection
is bounded by what `mk_proc_stat` exposes.

## Source map

```
  userland/capsule_process_manager/src/pm/app.rs      the App impl (manifest, event, paint, tick)
  userland/capsule_process_manager/src/pm/state.rs     the eight monitored rows + refresh
  userland/capsule_process_manager/src/pm/sample.rs    the mk_proc_stat CPU-delta sampling
  userland/capsule_process_manager/src/pm/paint.rs     the table and sparkline rendering
```

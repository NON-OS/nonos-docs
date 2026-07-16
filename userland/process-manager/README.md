# capsule_process_manager

`capsule_process_manager` is the NONOS task viewer: a small GUI window that lists the desktop
applications, shows whether each is running and under which pid, and paints a live CPU sparkline per row
sampled from the kernel. It is a read-only observer. It does not start, stop, or signal any process, and
it holds no authority over the processes it reports on.

It is an [app-skeleton](../writing-an-app.md) GUI app. The kernel spawns it under service handle
`app.process_manager` on service port 4730 with a reply port on 4731, and its capability mask is `0x1819`
(`userland/capsule_process_manager/Capsule.mk:11`). The source is `userland/capsule_process_manager/`.

## Contents

- [Overview](#overview)
- [Identity](#identity)
- [User reference](#user-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview

The process manager is an ordinary NONOS GUI application. Its entry point hands its `App` implementation
to the skeleton's `run`, so the runtime owns the surface, the window, the input subscription, and the
paint and tick loop, and the capsule supplies four things: a manifest for a normal window, an `on_event`
that refreshes on a key or click and closes on Escape, a `paint` that draws the table, and an `on_tick`
that periodically re-resolves the process list and samples the kernel for CPU ticks
(`userland/capsule_process_manager/src/main.rs:28`, `src/pm/app.rs:36`).

Unlike the service capsules, it runs no IPC server of its own and receives no application opcodes. It is
a poller: every tick it reads the kernel's process-statistics syscall and, every fifth tick, re-resolves
the monitored service names to pids through the skeleton's service lookup (`src/pm/app.rs:49`). What it
displays is exactly what those two reads expose, and nothing more.

## Identity

Everything the kernel and the service registry need to name and reach the capsule comes from its
`Capsule.mk` and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `process-manager` | `Capsule.mk:1` |
| Service handle | `app.process_manager` | `Capsule.mk:2`, `src/userspace/capsule_process_manager/spawn.rs:31` |
| Namespace | `systems.nonos.app.process_manager` | `Capsule.mk:7` |
| Service endpoint | `service:4730:app.process_manager` | `Capsule.mk:8`, `spawn.rs:32` |
| Reply endpoint | `reply:4731:endpoint.app.process_manager.reply` | `Capsule.mk:9`, `spawn.rs:33`, `spawn.rs:34` |
| Capability mask | `0x1819` | `Capsule.mk:11` |
| Binary name | `process_manager` | `Capsule.mk:5` |
| Feature gate | `nonos-capsule-process-manager` | `Capsule.mk:6` |
| Kernel mirror | `src/userspace/capsule_process_manager` | `Capsule.mk:12` |

The mask `0x1819` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x0001  CoreExec                bit()  1     types.rs:56
  0x0008  IPC                     bit()  8     types.rs:59
  0x0010  Memory                  bit() 16     types.rs:60
  0x0800  GraphicsDisplayQuery    bit() 2048   types.rs:67
  0x1000  GraphicsSurfaceCreate   bit() 4096   types.rs:68
  ------
  0x1819  = 1 + 8 + 16 + 2048 + 4096
```

The kernel spawn path requests exactly those five capabilities and no others
(`src/userspace/capsule_process_manager/spawn.rs:50`). The two graphics bits are the only difference
between this capsule and the service pools: it is a GUI app, so it needs to query the display and create a
surface to paint into, which is what those bits grant. There is no `Network` bit (4), no `FileSystem` bit
(64), no `Debug` bit (256), no `Admin` bit (512), and no hardware, driver, MMIO, IRQ, DMA, or PIO
capability. The whole basis of the security analysis below is that the capsule reads the process table
through an introspection syscall and creates a surface, and it can do nothing else.

## User reference

The window is a 440x240 normal window titled `Process Manager`, placed at (744, 456) and subscribed to
key-down input only (`src/pm/manifest.rs:19`, `manifest.rs:24`). It shows a title, a header row of
columns (`name`, `pid`, `caps`), one row per monitored application, a status line, and a refresh counter
(`src/pm/paint.rs:31`).

The monitored set is a fixed list of eight desktop applications, in this order: `terminal`,
`file_manager`, `text_editor`, `settings`, `process_manager`, `about`, `calculator`, and `desktop_shell`
(`src/pm/state.rs:42`). Each is a row bound to a service name (`app.terminal`, `app.file_manager`,
`app.text_editor`, `app.settings`, `app.process_manager`, `app.about`, `app.calculator`, and
`desktop_shell`) that the manager resolves to a live pid.

What each row shows:

| Column | Content | Source |
|---|---|---|
| name | the application label | `paint.rs:42` |
| pid | the resolved pid, or `offline` if the name did not resolve | `paint.rs:44`, `paint.rs:48` |
| caps | always `unavailable` (per-process capability reporting is not implemented) | `paint.rs:46`, `paint.rs:49` |
| sparkline | a 30-sample bar chart of recent CPU share | `paint.rs:51`, `paint.rs:59` |
| percent | the newest CPU-share sample as a number | `paint.rs:52` |

Below the table the manager draws a status line (`PID from service lookup, caps unavailable`) and a
`refreshes:` counter that increments on every refresh (`src/pm/paint.rs:35`, `paint.rs:38`,
`src/pm/state.rs:86`, `state.rs:75`).

Every action a user can take, and its handler:

| Action | Effect | Handler |
|---|---|---|
| Press Escape | close the window | `src/pm/event.rs:29` |
| Press any other key | force an immediate refresh (re-resolve pids) and repaint | `src/pm/event.rs:32` |
| Click in the window | force an immediate refresh and repaint | `src/pm/event.rs:22` |
| Wait | the tick loop refreshes and re-samples on its own | `src/pm/app.rs:49` |

There is no scroll, no sort, and no select or kill. The list is a fixed eight rows that fit the window,
the order is the static `KNOWN` order, and the capsule has no code path that signals or terminates a
process (there is no `kill` verb and no signalling syscall anywhere in the source). A keypress and a
mouse click do the same thing, which is to trigger a fresh service lookup so the pid column and the online
flag catch up immediately instead of waiting for the next automatic refresh.

### How it samples process data

The CPU column is the interesting mechanism. On every tick `sample` reads the kernel's
process-statistics syscall into a stack buffer sized for a header plus up to 64 entries, then computes
each monitored row's share of the total tick delta since the last sample (`src/pm/sample.rs:25`):

```
  sample(state):
      written = mk_proc_stat(buf, MAX_ENTRIES = 64)     // header{total_ticks,count} + entries{pid,run_ticks}
      if written <= 0: return
      dt = header.total_ticks - state.last_total_ticks
      warmed = state.last_total_ticks != 0 and dt > 0
      for each row with pid != 0:
          ticks = find_ticks(buf, row.pid)               // linear scan of the returned entries
          if warmed:
              d = ticks - state.last_ticks[row]
              pct = min(d * 100 / dt, 100)               // this pid's share of the interval
              row.cpu.percent[head] = pct; head = (head + 1) % HISTORY
          state.last_ticks[row] = ticks
      state.last_total_ticks = header.total_ticks
```

`mk_proc_stat` returns the number of per-process entries written and fills the buffer with a
`ProcStatHeader` carrying the system total tick count and an entry count, followed by one
`ProcStatEntry{pid, run_ticks}` per live pid (`userland/libc/src/procstat.rs:36`, `procstat.rs:21`,
`procstat.rs:30`). The manager caps the read at 64 entries (`src/pm/sample.rs:21`, `sample.rs:31`),
finds each monitored pid's run-tick count with a linear scan (`find_ticks`, `src/pm/sample.rs:54`),
takes the delta since the last sample, and expresses it as a percentage of the total delta, clamped to
100 (`sample.rs:44`). The first sample is not warmed, because there is no prior baseline, so a percentage
is only computed from the second sample onward (`sample.rs:35`). The result feeds a 30-slot circular
history that the sparkline renders (`src/pm/state.rs:19`, `src/pm/paint.rs:59`).

The pid itself comes from service discovery. On refresh, the manager walks the eight known rows and
resolves each service name to a pid through the skeleton's `lookup_service`, marking the row online and
recording the pid when the lookup succeeds and offline when it does not (`src/pm/state.rs:74`,
`state.rs:81`). A row whose service has crashed and not yet respawned reads offline until the next
refresh, which is at most a few ticks away, or immediately if the user presses a key or clicks.

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` calls the skeleton's `run(ProcessManager::new)`
(`src/main.rs:27`). The single module tree is `pm`, split one unit per file: `app` (the `App` impl),
`event` (the input handler), `manifest` (the window shape), `paint` (the renderer), `sample` (the CPU
sampler), `state` (the monitored rows and refresh), `format` (a `u32`-to-decimal helper), and `theme`
(the colours) (`src/pm/mod.rs:17`).

The model is a `ProcessManager` holding a `State` and a tick counter (`src/pm/app.rs:25`). The `State`
holds the eight rows, a refresh counter, a status string, the last total-tick snapshot, and the per-row
last-tick snapshots that drive the CPU delta (`src/pm/state.rs:53`). Each `Row` carries a label, a
service name, a pid, an online flag, and a 30-sample circular CPU history (`src/pm/state.rs:31`).

Lifecycle:

1. The kernel spawns the capsule at boot through the apps-and-tools fleet plan
   (`src/userspace/init/spawn_plan/apps_tools.rs:50`, behind the `nonos-capsule-process-manager`
   feature), which calls `super::boot::capsule("APP-PROCESS-MANAGER", "app_process_manager", ...)`
   (`apps_tools.rs:52`). The spawn path verifies the embedded ELF, id cert, manifest, and ZK attestation
   trailer, registers `app.process_manager` on port 4730 with its reply inbox on 4731, and on success the
   boot log prints `[APP-PROCESS-MANAGER] capsule spawned`
   (`src/userspace/capsule_process_manager/spawn.rs:37`, `src/userspace/init/capsule_boot/run.rs:29`).
2. The skeleton `run` initialises the heap, resolves the desktop peers, creates the window from the
   manifest, and drives the event and tick loop (`userland/app_skeleton/src/runner/entry.rs:31`).
3. `on_tick` fires on the skeleton's timer (a 1000 ms default interval, `app/behavior.rs:30`). Every
   fifth tick it calls `state.refresh()` to re-resolve pids; every tick it calls `sample()` to read
   `mk_proc_stat` and update the CPU history, then returns `true` to request a repaint (`src/pm/app.rs:49`).
4. `on_event` handles input between ticks: a pointer button-down or any non-Escape key-down triggers a
   refresh and a repaint, and Escape closes the window (`src/pm/event.rs:21`).
5. `paint` projects the `State` into the surface: the title, the column headers, a row per application,
   the status line, the refresh counter, and a sparkline per row (`src/pm/paint.rs:29`). The frame lands
   in the shared surface the compositor presents.

## Protocol and IPC

The process manager exposes no application opcodes of its own beyond what the app skeleton registers for
it (the `app.process_manager` service on port 4730 and the reply inbox on 4731,
`src/userspace/capsule_process_manager/spawn.rs:31`). Everything it does that reaches outside the capsule
is one of two reads.

Process statistics, through `mk_proc_stat` (`userland/libc/src/procstat.rs:36`), a direct syscall
`N_MK_PROC_STAT` with no service on the far side. It lands in the kernel at `sys_proc_stat`
(`src/syscall/microkernel/procstat.rs:43`), which enumerates all live pids (`procstat.rs:44`), writes a
`ProcStatHeader{total_ticks, count}` seeded from the timer tick count (`procstat.rs:53`), then one
`ProcStatEntry{pid, run_ticks}` per pid up to the caller's `max_entries`, with `run_ticks` taken from the
scheduler's per-pid tick accounting (`procstat.rs:62`, `procstat.rs:67`). A NULL buffer or zero
`max_entries` probes the count without copying (`procstat.rs:45`). The kernel validates the destination
against the caller's address space before every write and returns `ERRNO_FAULT` on a bad buffer
(`procstat.rs:50`, `procstat.rs:58`). The syscall returns the number of entries written (`procstat.rs:74`).

Service discovery, through `lookup_service` (`userland/app_skeleton/src/discover/lookup_service.rs:21`),
which wraps `mk_service_lookup` and returns the resolved port and pid or `None`
(`lookup_service.rs:24`). The manager uses only the pid (`src/pm/state.rs:82`).

That is the entire external surface. There are no VFS calls, no network calls, no clipboard, and no
installer path.

## Security analysis

The process manager is the capsule that shows you the whole process table, and it holds no authority over
a single row of it. Its mask `0x1819` grants CoreExec, IPC, Memory, GraphicsDisplayQuery, and
GraphicsSurfaceCreate (`Capsule.mk:11`, `src/userspace/capsule_process_manager/spawn.rs:50`), and nothing
else.

Its one introspection power is the `mk_proc_stat` syscall, and that syscall is a read. It exposes a
system-wide list of live pids and their run-tick counts (`src/syscall/microkernel/procstat.rs:44`), but
it copies data out and takes nothing in beyond a buffer and a length; there is no pid argument that would
let a caller act on a process, and the kernel bounds every write to the caller's own address space
(`procstat.rs:50`). What lets the capsule read the whole table is not a per-process capability but the
plain existence of the syscall, which any capsule can call; the manager holds no privileged bit for it.
Note that the syscall does not gate on the `Debug` capability, and the manager does not hold `Debug`
anyway (256 is absent from `0x1819`), so process introspection here is a public read, not a privileged
one.

It cannot kill or signal. There is no signalling syscall in the capsule, no `kill` action in the user
reference, and no `Admin` bit (512) in the mask, so the capsule cannot reboot, terminate, or send a
signal to any process. It cannot touch hardware: no Driver, Mmio, Irq, Dma, or Pio. It cannot open a file
or a socket: no FileSystem, no Network. Its isolation is that it never holds a per-process handle. It
holds only pids it resolved by name through the service registry (`src/pm/state.rs:81`) and tick counts it
read from the introspection syscall, both of which are values, not handles. A bug in the sampler or the
renderer cannot escalate past a read of the process table, because a read is all the capsule ever had.

The one honest boundary the tool states about itself is the `caps` column: it renders a hardcoded
`unavailable`, because per-process capability reporting is not implemented, so the capsule that decodes
capability masks in this page cannot in fact show the mask of any process it lists (`src/pm/paint.rs:46`,
`paint.rs:49`).

## How to contribute

The source lives at `userland/capsule_process_manager/`. The whole capsule is under `src/pm/`, one unit
per file: the `App` glue in `app.rs`, the input handler in `event.rs`, the window shape in `manifest.rs`,
the renderer in `paint.rs`, the CPU sampler in `sample.rs`, the monitored rows and refresh in `state.rs`,
the decimal helper in `format.rs`, and the palette in `theme.rs`.

To change what the tool watches, edit the `KNOWN` table in `src/pm/state.rs:42`: each entry is a label
and the service name to resolve, and the array length and the `State` fields (`rows`, `last_ticks`) are
fixed at eight, so add a row there and keep the widths in step (`src/pm/state.rs:54`, `state.rs:58`). To
change the sampling window or math, edit `src/pm/sample.rs` (the `MAX_ENTRIES` cap and the delta
computation) and `HISTORY` in `src/pm/state.rs:19`. To change the layout, edit `src/pm/paint.rs` and the
column offsets at its head (`paint.rs:24`).

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
included through `userland/capsule_process_manager/Capsule.mk:14`):

```
  make nonos-mk-process-manager                 build the capsule ELF
  make nonos-mk-process-manager-sign            produce the id cert, manifest, and attestation trailer
  make nonos-mk-process-manager-verify          verify the signed artifacts against the trust anchor
  make nonos-mk-check-process-manager-keys      check the per-capsule signing keys exist
```

For a running desktop that includes the process manager, `make nonos-mk-process-manager-prod` builds the
full desktop GUI image (`Makefile:1187`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (the sampler returns early on a bad read rather than trusting it,
`src/pm/sample.rs:28`, and the release profile is `panic = "abort"`, `Cargo.toml:26`); modular files, one
unit per file, with `mod.rs` used only for re-exports (`src/pm/mod.rs:17`); and the AGPL header at the top
of every source file, matching the header on every existing module.

## Debugging

The process manager has no IPC server and no service port to receive on: it is an app-skeleton
application on the app port `app.process_manager` :4730 that polls the kernel on a timer. Because it is an
observer rather than a server, its failure signatures are visual rather than errnos.

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[APP-PROCESS-MANAGER] capsule spawned` (tag `APP-PROCESS-MANAGER`, message `capsule spawned`) from the
boot log (`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). An absent line
means the capsule never started, usually a signature, manifest, or attestation failure; the error path
prints an `[ERROR]` line with the specific `SpawnError` instead (`src/userspace/init/capsule_boot/run.rs:32`,
`src/sys/boot_log/output.rs:49`).

Failure modes and where to look:

- A row reads `offline`. `lookup_service` did not resolve that service name to a pid on the last refresh
  (`src/pm/state.rs:81`, `userland/app_skeleton/src/discover/lookup_service.rs:25`). For a service that
  just crashed this lags the truth by up to one refresh interval (a few ticks); pressing any key or
  clicking forces an immediate re-resolve (`src/pm/event.rs:32`). A window that opens but shows every row
  offline points at the service registry or the monitored apps not being up, not at the manager itself.
- A flat or blank sparkline. The sampler has not warmed yet: a percentage is only computed from the
  second `mk_proc_stat` sample onward, when there is a prior baseline (`src/pm/sample.rs:35`). A row also
  reads flat while its pid is zero, because an offline row is skipped in the sampler (`sample.rs:37`).
- The CPU column stays zero for a live row. `mk_proc_stat` returned `<= 0` and the sampler bailed
  (`src/pm/sample.rs:28`), or the row's pid was not among the first 64 entries the read capped at
  (`sample.rs:31`, `sample.rs:54`). The refresh counter still advances because it is driven by
  `refresh()`, not by the sampler (`src/pm/state.rs:75`).
- The `caps` column always says `unavailable`. That is by design, not a fault: per-process capability
  reporting is not implemented (`src/pm/paint.rs:46`).

## Source map

```
  src/main.rs                              _start -> run(ProcessManager::new)
  src/pm/mod.rs                            the module tree (app, event, manifest, paint, sample, state, format, theme)
  src/pm/app.rs                            the App impl (manifest, event, paint, tick)
  src/pm/event.rs                          the input handler (key/click refresh, Esc close)
  src/pm/manifest.rs                       the window shape (440x240, key-down subscription)
  src/pm/state.rs                          the eight monitored rows, refresh, service lookup
  src/pm/sample.rs                         the mk_proc_stat CPU-delta sampling
  src/pm/paint.rs                          the table and sparkline rendering
  src/pm/format.rs                         the u32-to-decimal helper
  src/pm/theme.rs                          the palette
  Capsule.mk                               slug, handle, ports, capability mask, kernel mirror
  userland/libc/src/procstat.rs            mk_proc_stat and the ProcStat header/entry layout
  userland/app_skeleton/src/discover/      lookup_service and the ServicePeer type
  src/userspace/capsule_process_manager/   the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/apps_tools.rs   the apps-and-tools fleet spawn entry
  src/syscall/microkernel/procstat.rs      the kernel sys_proc_stat handler
  nonos-mk/capsule.mk                      the generated nonos-mk-process-manager[-sign|-verify] targets
```

Every reference above is verified against those trees.

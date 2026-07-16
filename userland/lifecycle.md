# Userland Lifecycle and Launch

This page describes how userland capsules are spawned, registered, tracked, and
launched after boot. Read [Userland Model](README.md), [Protocol Atlas](protocols.md),
and [GUI Contracts](gui-contracts.md) first.

The lifecycle code is intentionally small. It tracks pid and generation, checks
liveness against the process table, and lets the init loop keep that state
current. It does not imply that every capsule is auto-restarted by init today.

---

## 1. Init sequence

Init starts in `run_init`. The ordered sequence is user-entry proof, std proof,
ripgrep, RAMFS, core after RAMFS, display core, drivers, VFS, network, desktop,
market, apps, then the supervisor loop after init lowers its own priority and
launches the final payload (`src/userspace/init/entry.rs:22`,
`src/userspace/init/entry.rs:25`, `src/userspace/init/entry.rs:26`,
`src/userspace/init/entry.rs:27`, `src/userspace/init/entry.rs:28`,
`src/userspace/init/entry.rs:29`, `src/userspace/init/entry.rs:30`,
`src/userspace/init/entry.rs:31`, `src/userspace/init/entry.rs:32`,
`src/userspace/init/entry.rs:33`, `src/userspace/init/entry.rs:35`,
`src/userspace/init/entry.rs:37`, `src/userspace/init/entry.rs:38`).

```
+------------------+
| run_init         |
+--------+---------+
         |
+--------+---------+
| proof and ramfs  |
+--------+---------+
         |
+--------+---------+
| core and drivers |
+--------+---------+
         |
+--------+---------+
| vfs and network  |
+--------+---------+
         |
+--------+---------+
| desktop and apps |
+--------+---------+
         |
+--------+---------+
| supervisor loop  |
+------------------+
```

The spawn planner keeps the main entry point small. Core after RAMFS spawns
keyring, entropy, crypto, and policy (`src/userspace/init/spawn_plan/core.rs:22`).
Driver startup is grouped as virtio, bus, input, NIC, USB, and storage
(`src/userspace/init/spawn_plan/orchestrator.rs:29`). Network startup calls L2,
IP, UDP, DHCP, TCP, DNS, Nym, and sockets in that order
(`src/userspace/init/spawn_plan/network.rs:17`). Desktop startup calls GUI core,
WM, wallpaper catalog, wallpaper, shell, and desktop services
(`src/userspace/init/spawn_plan/desktop_fleet.rs:17`). Desktop services are image
codec, clipboard, attest, login, and toolkit
(`src/userspace/init/spawn_plan/desktop_services.rs:17`).

When `microkernel-input-probe` is enabled, desktop startup uses the input probe
fleet instead of the normal desktop fleet (`src/userspace/init/spawn_plan/orchestrator.rs:46`).
When `microkernel-setup-wizard` is enabled without input probe, init starts GUI
core and setup wizard first, and `spawn_post_wizard` later starts the normal
desktop fleet and market (`src/userspace/init/spawn_plan/orchestrator.rs:51`,
`src/userspace/init/spawn_plan/orchestrator.rs:63`).

## 2. Verified spawn contract

`CapsuleSpecVerified` is the kernel-side signed capsule contract. It contains
service name, service port, reply inbox, reply port, ELF bytes, NØNOS-ID
certificate bytes, manifest bytes, target triple, requested capability ceiling,
and debug tag (`src/kernel_core/process_spawn/capsule_spawn/spec.rs:31`).

Preflight decodes the NØNOS-ID certificate, verifies it against the production
policy and trust anchor, declares the service and reply endpoints, and verifies
the manifest against certificate, verified identity, trust anchor, ELF, target
triple, requested caps, and declared endpoints
(`src/kernel_core/process_spawn/capsule_spawn/runner/preflight.rs:29`,
`src/kernel_core/process_spawn/capsule_spawn/runner/preflight.rs:34`,
`src/kernel_core/process_spawn/capsule_spawn/runner/preflight.rs:36`,
`src/kernel_core/process_spawn/capsule_spawn/runner/preflight.rs:39`,
`src/kernel_core/process_spawn/capsule_spawn/runner/preflight.rs:48`).

Installed process capabilities come from the verified manifest result, not from
the raw requested cap mask. The spawn wrapper passes `preflighted.install_caps`
into install (`src/kernel_core/process_spawn/capsule_spawn/runner/verified.rs:23`,
`src/kernel_core/process_spawn/capsule_spawn/runner/verified.rs:31`,
`src/kernel_core/process_spawn/capsule_spawn/runner/verified.rs:38`).

Install registers the reply inbox and reply endpoint, creates the process,
records the reply inbox on the PCB, registers the per-process inbox, loads the
ELF, installs caps, allocates kernel and user stacks, builds the first user
context, registers the service endpoint, logs the spawn, and adds the pid to the
runqueue (`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:35`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:36`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:38`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:40`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:42`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:44`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:46`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:48`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:49`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:50`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:51`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:53`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:54`).

```
+--------------------+
| CapsuleSpecVerified|
+---------+----------+
          |
+---------+----------+
| id cert verify     |
| manifest verify    |
+---------+----------+
          |
+---------+----------+
| install_caps       |
+---------+----------+
          |
+---------+----------+
| create process     |
| load elf           |
| install caps       |
+---------+----------+
          |
+---------+----------+
| service endpoint   |
| runqueue           |
+--------------------+
```

## 3. Lifecycle state

Each tracked capsule exposes a static `CapsuleState`. The desktop shell and
terminal mirrors show the pattern: a static state, a private `set_alive(pid)`,
and a public `shared_state()` accessor
(`src/userspace/capsule_desktop_shell/state.rs:17`,
`src/userspace/capsule_desktop_shell/state.rs:19`,
`src/userspace/capsule_desktop_shell/state.rs:21`,
`src/userspace/capsule_desktop_shell/state.rs:25`,
`src/userspace/capsule_terminal/state.rs:17`,
`src/userspace/capsule_terminal/state.rs:19`,
`src/userspace/capsule_terminal/state.rs:21`,
`src/userspace/capsule_terminal/state.rs:25`).

`CapsuleState` stores pid, generation, restart count, last exit timestamp, max
restart count, and respawn debounce (`src/services/lifecycle/state/types.rs:21`).
The default restart cap is `8`, and the default respawn debounce is `2000`
milliseconds (`src/services/lifecycle/state/constants.rs:17`,
`src/services/lifecycle/state/constants.rs:18`).

`set_alive` stores the pid and increments generation. `is_alive` rejects pid
zero, checks the process table, accepts `New`, `Ready`, `Running`, `Sleeping`,
and `Stopped` as live states, and clears the pid when the process is gone or no
longer live (`src/services/lifecycle/state/liveness.rs:23`,
`src/services/lifecycle/state/liveness.rs:36`,
`src/services/lifecycle/state/liveness.rs:41`,
`src/services/lifecycle/state/liveness.rs:43`,
`src/services/lifecycle/state/liveness.rs:51`,
`src/services/lifecycle/state/liveness.rs:56`).

```
+--------------------------+
| set alive pid            |
+------------+-------------+
             |
+------------+-------------+
| store pid                |
| advance generation       |
+------------+-------------+
             |
+------------+-------------+
| is alive check           |
| process table lookup     |
+------------+-------------+
             |
+------------+-------------+
| live state keeps pid     |
| dead state clears pid    |
+--------------------------+
```

The registry stores name and state pairs. A successful init boot helper logs the
spawn and registers the capsule; a failed spawn logs the error and does not
register (`src/userspace/init/capsule_boot/run.rs:21`,
`src/userspace/init/capsule_boot/run.rs:27`,
`src/userspace/init/capsule_boot/run.rs:29`,
`src/userspace/init/capsule_boot/run.rs:30`,
`src/userspace/init/capsule_boot/run.rs:32`). Registry `tick` iterates registered
capsules and calls `state.is_alive()` (`src/services/lifecycle/registry.rs:35`,
`src/services/lifecycle/registry.rs:46`,
`src/services/lifecycle/registry.rs:48`,
`src/services/lifecycle/registry.rs:49`).

The lifecycle transport captures generation before send, rejects dead capsules
before enqueue, enqueues to `proc.<pid>`, wakes the sleeping owner when needed,
rechecks liveness and generation while waiting for the reply, rejects stale
generation, ignores replies with the wrong request id, and yields between polls
(`src/services/lifecycle/transport.rs:134`,
`src/services/lifecycle/transport.rs:142`,
`src/services/lifecycle/transport.rs:143`,
`src/services/lifecycle/transport.rs:146`,
`src/services/lifecycle/transport.rs:149`,
`src/services/lifecycle/transport.rs:151`,
`src/services/lifecycle/transport.rs:169`,
`src/services/lifecycle/transport.rs:170`,
`src/services/lifecycle/transport.rs:173`,
`src/services/lifecycle/transport.rs:180`,
`src/services/lifecycle/transport.rs:181`,
`src/services/lifecycle/transport.rs:186`).

```
+--------------------------+
| lifecycle transport send |
+------------+-------------+
             |
+------------+-------------+
| capture generation       |
| reject dead capsule      |
+------------+-------------+
             |
+------------+-------------+
| enqueue proc pid inbox   |
| wake sleeping owner      |
+------------+-------------+
             |
+------------+-------------+
| wait for reply           |
| recheck generation       |
+------------+-------------+
             |
+------------+-------------+
| matching request id      |
| response returned        |
+--------------------------+
```

There is a generic supervised respawn helper. It skips live entries, skips never
spawned entries, checks `should_respawn`, records exit time, and calls the spawn
function (`src/services/lifecycle/supervisor.rs:25`,
`src/services/lifecycle/supervisor.rs:29`,
`src/services/lifecycle/supervisor.rs:32`,
`src/services/lifecycle/supervisor.rs:35`,
`src/services/lifecycle/supervisor.rs:38`,
`src/services/lifecycle/supervisor.rs:39`). The init loop currently calls the
registry tick, not `supervisor_tick`; `supervisor_tick` is only re-exported from
the lifecycle module (`src/userspace/init/supervisor/loop_impl.rs:25`,
`src/userspace/init/supervisor/loop_impl.rs:26`,
`src/services/lifecycle/mod.rs:38`).

## 4. Init supervisor loop

After boot, init lowers its priority, yields, launches the final payload, and
enters the supervisor loop (`src/userspace/init/entry.rs:35`,
`src/userspace/init/entry.rs:36`, `src/userspace/init/entry.rs:37`,
`src/userspace/init/entry.rs:38`).

The loop ticks lifecycle once every 1000 milliseconds, conditionally starts the
post-wizard desktop after the setup wizard dies, then yields
(`src/userspace/init/supervisor/loop_impl.rs:23`,
`src/userspace/init/supervisor/loop_impl.rs:31`,
`src/userspace/init/supervisor/loop_impl.rs:32`,
`src/userspace/init/supervisor/loop_impl.rs:35`,
`src/userspace/init/supervisor/loop_impl.rs:40`). It does not drain any
launcher inbox; the kernel does not probe capsules, and a capsule that exited is
observed `Dead` on its next IPC rather than by an active poll.

```
+----------------------+
| init supervisor loop |
+----------+-----------+
           |
+----------+-----------+
| time >= last + 1000  |
| lifecycle tick       |
+----------+-----------+
           |
+----------+-----------+
| optional post wizard |
+----------+-----------+
           |
+----------+-----------+
| yield scheduler      |
+----------------------+
```

## 5. Desktop launcher lifecycle

There is no launch broker in the current tree. The apps are spawned once at boot
by `spawn_apps`, so a taskbar click never has to start anything
(`src/userspace/init/spawn_plan/apps.rs:17`). The desktop shell taskbar carries
nine launcher entries, each with an icon, a label, and a service name
(`userland/capsule_desktop_shell/src/state/apps.rs:36`).

A click resolves to `launcher_request::request`, which looks up the app's
service pid through `mk_service_lookup`. If the service resolves to a live pid,
the shell sends that pid an eight-byte `NCTL` focus frame through
`mk_ipc_send_to_pid` and the click returns success. If the lookup fails, the
request returns false and the click does nothing, since the target app is either
not spawned or has exited
(`userland/capsule_desktop_shell/src/server/handlers/launcher_request.rs:26`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_request.rs:32`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_request.rs:42`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_focus.rs:24`).

```
+------------------+
| launcher click   |
+--------+---------+
         |
+--------+---------+
| service lookup   |
+---+----------+---+
    |          |
    | pid      | no pid
    |          |
+---+---+  +---+------+
| NCTL  |  | ignored  |
| focus |  |          |
+-------+  +----------+
```

## 6. Direct GUI proofs

The input probe and setup wizard are not ordinary first-party app launcher
entries. The input probe initializes heap, runs setup, then enters its direct
server runner (`userland/capsule_input_probe/src/main.rs:16`,
`userland/capsule_input_probe/src/main.rs:20`,
`userland/capsule_input_probe/src/main.rs:24`). Its runner subscribes to the
input router, grabs keyboard, blocks on inbox `0`, parses delivery frames, and
draws printable key-down events (`userland/capsule_input_probe/src/server/runner.rs:12`,
`userland/capsule_input_probe/src/server/runner.rs:13`,
`userland/capsule_input_probe/src/server/runner.rs:14`,
`userland/capsule_input_probe/src/server/runner.rs:18`,
`userland/capsule_input_probe/src/server/runner.rs:28`,
`userland/capsule_input_probe/src/server/runner.rs:31`,
`userland/capsule_input_probe/src/server/runner.rs:37`).

The setup wizard subscribes to input, grabs keyboard, draws an initial screen,
reads delivery frames from inbox `0`, processes only key-down events, advances
its step state, removes the compositor scene on completion, and exits
(`userland/capsule_setup_wizard/src/server/runner.rs:12`,
`userland/capsule_setup_wizard/src/server/runner.rs:13`,
`userland/capsule_setup_wizard/src/server/runner.rs:14`,
`userland/capsule_setup_wizard/src/server/runner.rs:15`,
`userland/capsule_setup_wizard/src/server/runner.rs:19`,
`userland/capsule_setup_wizard/src/server/runner.rs:23`,
`userland/capsule_setup_wizard/src/server/runner.rs:26`,
`userland/capsule_setup_wizard/src/server/runner.rs:29`,
`userland/capsule_setup_wizard/src/server/runner.rs:30`,
`userland/capsule_setup_wizard/src/server/runner.rs:31`,
`userland/capsule_setup_wizard/src/server/runner.rs:32`,
`userland/capsule_setup_wizard/src/server/runner.rs:33`).

## 7. Exit and grant release

The lifecycle has three points, spawn, run, and exit, and exit is the one that makes
the capability model safe over time. When a capsule exits, teardown releases every
hardware grant the pid still held: it calls `release_all_for_pid` for the broker claim
table, then the IRQ, DMA, and (on x86_64) PIO release-all paths, so a dying capsule
cannot leave an interrupt bound, a DMA buffer pinned, or a device claimed
(`src/process/exit/teardown.rs:33`). The later finalize pass repeats the broker releases
and additionally unregisters the capsule's service endpoints and its per-process inbox
(`src/process/exit/finalize.rs:18`, `src/process/exit/finalize.rs:23`). The order matters:
the claim table's `release_all_for_pid` retains only the claims not held by the exiting
pid, which is what closes the window where a released device could be reclaimed while an
old grant handle still names it (the claim and epoch model is on
[the hardware broker page](../subsystems/hardware-broker/claim.md)).

The consequence for the system is that authority is not just granted at spawn from the
verified manifest; it is fully reclaimed at exit by the kernel, without the capsule's
cooperation. A capsule that crashes, is killed, or exits cleanly all reach the same
teardown, so a driver that panics does not strand its device claim, and a service that
exits does not leave a dangling endpoint another capsule could resolve.

## 8. Security analysis

The lifecycle enforces three properties that together make the capsule model safe across
spawn, run, and exit. The first is that nothing runs unverified: preflight verifies the
NØNOS-ID certificate and the manifest against the ELF, the trust anchor, the target
triple, and the requested caps before install loads the binary
(`src/kernel_core/process_spawn/capsule_spawn/runner/preflight.rs:29`). The installed caps
come from the verified manifest, never from the raw requested mask
(`src/kernel_core/process_spawn/capsule_spawn/runner/verified.rs:23`), so a capsule cannot
widen its authority at the spawn site.

The second is that no capsule spawns another. Every capsule that runs was spawned by init
through the verified path, including the app fleet, which `spawn_apps` starts at boot
(`src/userspace/init/spawn_plan/apps.rs:17`). A taskbar click in the desktop shell does not
spawn anything; it resolves the app's service name to a pid and, if that pid exists, sends
it an `NCTL` focus frame (`userland/capsule_desktop_shell/src/server/handlers/launcher_request.rs:26`).
So the shell can focus an already-running app but cannot mint a new process, which keeps the
spawn authority entirely inside init and behind verification.

The third is that liveness is checked against the real process table, not a cached flag.
`is_alive` rejects pid zero, looks the pid up in the process table, accepts `New`, `Ready`,
`Running`, `Sleeping`, and `Stopped` as live, and clears the recorded pid when the process
is gone (`src/services/lifecycle/state/liveness.rs:36`). The lifecycle transport that talks
to a supervised capsule captures the generation before it sends
(`src/services/lifecycle/transport.rs:159`), rejects a dead capsule before enqueue
(`src/services/lifecycle/transport.rs:160`), and rechecks liveness and generation while
waiting for the reply, rejecting a stale generation
(`src/services/lifecycle/transport.rs:187`, `src/services/lifecycle/transport.rs:190`). The
generation counter is the anti-stale mechanism here: a reply from a re-spawned capsule
carrying an old generation is refused, so a message cannot be answered by the wrong
incarnation of a service.

## 9. Debugging the lifecycle

The first marker for any capsule is the boot line: the init boot helper logs `capsule
spawned` and registers the capsule on success, and logs the mapped error string without
registering on failure (`src/userspace/init/capsule_boot/run.rs:29`,
`src/userspace/init/capsule_boot/run.rs:32`). A capsule that never logged `capsule
spawned` failed at spawn, and the error string names the stage, from `capsule binary not
embedded (feature off)` through the certificate and manifest rejection reasons
(`src/userspace/init/capsule_boot/error.rs:21`, `src/kernel_core/process_spawn/capsule_spawn/spec.rs:48`).
A capsule that logged the marker but is no longer present exited at runtime, and the
registry tick is what observes its pid went away: the init loop calls
`crate::services::lifecycle::tick` once per second, which walks the registered capsules and
calls `is_alive` on each (`src/userspace/init/supervisor/loop_impl.rs:32`,
`src/services/lifecycle/registry.rs:46`).

Two boot markers exist and come from different points in the spawn path. The spawn
primitive prints `[SPAWN] name=<name> pid=... caps=<mask> entry=...` for every capsule as it
installs the ELF, so this line appears once for anything that spawns at all
(`src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs:52`,
`src/kernel_core/process_spawn/capsule_spawn/runner/install/spawn_log.rs:18`). On top of
that, the init boot helper prints an extra `[<PREFIX>] capsule spawned` line for every
capsule it brings up through `boot::capsule`, which is the ramfs, core services, drivers,
desktop fleet, and boot apps (`src/userspace/init/capsule_boot/run.rs:29`). So a boot-fleet
capsule shows both lines, while a capsule spawned outside the init boot helper shows only
the `[SPAWN] name=...` line. When you are checking whether an init-spawned capsule ran, the
`[<PREFIX>] capsule spawned` line is the one to grep for.

An important honesty note the code carries: the init loop ticks the lifecycle registry, it
does not call `supervisor_tick`. The generic supervised-respawn helper exists and the state
carries a restart cap of `8` and a `2000` millisecond respawn debounce
(`src/services/lifecycle/state/constants.rs:17`, `src/services/lifecycle/state/constants.rs:18`),
but the init loop calls the registry tick rather than the respawn helper, which is only
re-exported from the lifecycle module (`src/services/lifecycle/mod.rs:37`,
`src/services/lifecycle/supervisor.rs:25`). So a capsule that exits is observed as dead and
its pid is cleared, but it is not automatically restarted by init today. Debugging a
service that "should come back" starts here: it will not, unless something explicitly
re-spawns it.

A taskbar click that does nothing is the service lookup, not a launch failure. On this
branch the apps are already running from boot, so the click path only resolves the app's
service name and sends the pid an `NCTL` focus frame; if the lookup returns no pid, nothing
is sent (`userland/capsule_desktop_shell/src/server/handlers/launcher_request.rs:26`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_request.rs:27`). So a dead
taskbar entry means the app it points at was never spawned or has since exited, which loops
back to the `capsule spawned` marker for that app.

## 10. Source map

```
  src/userspace/init/entry.rs                        run_init and the ordered spawn sequence
  src/userspace/init/spawn_plan/{core,drivers_*,network/,desktop_fleet,apps}.rs  per-phase spawn
  src/userspace/init/supervisor/loop_impl.rs          the init residual loop (registry tick)
  src/userspace/init/capsule_boot/{run,error}.rs      the boot marker and error strings
  src/kernel_core/process_spawn/capsule_spawn/runner/  preflight, verified, install
  src/services/lifecycle/{state/,registry.rs,transport.rs,supervisor.rs}  liveness and transport
  src/process/exit/{teardown,finalize}.rs             grant release and endpoint unregister at exit
  userland/capsule_desktop_shell/src/server/handlers/launcher_{request,focus}.rs  taskbar focus
```

The verified spawn gate is also summarized on [the userland model page](README.md); the
claim and epoch model that exit release feeds into is on
[the hardware broker page](../subsystems/hardware-broker/claim.md).

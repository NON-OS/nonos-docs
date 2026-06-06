# Userland Lifecycle and Launch

This page describes how userland capsules are spawned, registered, tracked, and
launched after boot. Read [Userland Model](README.md), [Protocol Atlas](protocols.md),
and [GUI Contracts](gui-contracts.md) first.

The lifecycle code is intentionally small. It tracks pid and generation, checks
liveness against the process table, and lets the init loop keep that state
current. It does not imply that every capsule is auto-restarted by init today.

---

## 1. Init sequence

Init starts in `run_init`. The ordered sequence is user-entry proof, RAMFS,
RAMFS smoke test, core services, drivers, VFS, network, launcher registration,
desktop, market, smoke tests, final payload, then the supervisor loop
(`src/userspace/init/entry.rs:25`, `src/userspace/init/entry.rs:27`,
`src/userspace/init/entry.rs:28`, `src/userspace/init/entry.rs:29`,
`src/userspace/init/entry.rs:30`, `src/userspace/init/entry.rs:31`,
`src/userspace/init/entry.rs:32`, `src/userspace/init/entry.rs:33`,
`src/userspace/init/entry.rs:34`, `src/userspace/init/entry.rs:35`,
`src/userspace/init/entry.rs:36`, `src/userspace/init/entry.rs:37`,
`src/userspace/init/entry.rs:41`, `src/userspace/init/entry.rs:42`).

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
| launcher broker  |
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
enters the supervisor loop (`src/userspace/init/entry.rs:39`,
`src/userspace/init/entry.rs:40`, `src/userspace/init/entry.rs:41`,
`src/userspace/init/entry.rs:42`).

The loop ticks lifecycle once every 1000 milliseconds, drains desktop launcher
requests every iteration, conditionally starts the post-wizard desktop after the
setup wizard dies, then yields (`src/userspace/init/supervisor/loop_impl.rs:17`,
`src/userspace/init/supervisor/loop_impl.rs:19`,
`src/userspace/init/supervisor/loop_impl.rs:24`,
`src/userspace/init/supervisor/loop_impl.rs:25`,
`src/userspace/init/supervisor/loop_impl.rs:26`,
`src/userspace/init/supervisor/loop_impl.rs:29`,
`src/userspace/init/supervisor/loop_impl.rs:31`,
`src/userspace/init/supervisor/loop_impl.rs:34`,
`src/userspace/init/supervisor/loop_impl.rs:37`).

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
| drain launcher       |
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

The launcher broker is registered before desktop startup, so the desktop shell
can discover it after boot (`src/userspace/init/entry.rs:34`,
`src/userspace/init/entry.rs:35`). Registration creates a kernel-owned inbox and
registers service `desktop.launcher` on port `4700` with IPC capability
(`src/userspace/init/launcher/register.rs:19`,
`src/userspace/init/launcher/register.rs:20`,
`src/userspace/init/launcher/register.rs:21`,
`src/userspace/init/launcher/register.rs:23`,
`src/userspace/init/launcher/register.rs:24`,
`src/userspace/init/launcher/register.rs:25`).

The broker drains messages from the existing launcher inbox, authorizes the
source, decodes the 8-byte request, and calls the spawn allowlist
(`src/userspace/init/launcher/drain.rs:17`,
`src/userspace/init/launcher/drain.rs:18`,
`src/userspace/init/launcher/drain.rs:21`,
`src/userspace/init/launcher/drain.rs:24`,
`src/userspace/init/launcher/drain.rs:27`). Authorization accepts only messages
whose `from` field is `proc.<pid>` and whose pid matches the registered
`desktop_shell` service (`src/userspace/init/launcher/authorize.rs:19`,
`src/userspace/init/launcher/authorize.rs:20`,
`src/userspace/init/launcher/authorize.rs:23`,
`src/userspace/init/launcher/authorize.rs:26`,
`src/userspace/init/launcher/authorize.rs:29`).

The allowlist maps ids `1` through `7` to terminal, file manager, text editor,
settings, process manager, about, and calculator. Each id returns true if the
capsule is already alive, otherwise it calls that capsule's verified spawn
wrapper (`src/userspace/init/launcher/spawn.rs:17`,
`src/userspace/init/launcher/spawn.rs:19`,
`src/userspace/init/launcher/spawn.rs:20`,
`src/userspace/init/launcher/spawn.rs:21`,
`src/userspace/init/launcher/spawn.rs:22`,
`src/userspace/init/launcher/spawn.rs:23`,
`src/userspace/init/launcher/spawn.rs:24`,
`src/userspace/init/launcher/spawn.rs:25`,
`src/userspace/init/launcher/spawn.rs:26`,
`src/userspace/init/launcher/spawn.rs:27`,
`src/userspace/init/launcher/spawn.rs:28`,
`src/userspace/init/launcher/spawn.rs:29`,
`src/userspace/init/launcher/spawn.rs:30`,
`src/userspace/init/launcher/spawn.rs:31`,
`src/userspace/init/launcher/spawn.rs:32`,
`src/userspace/init/launcher/spawn.rs:33`).

Desktop shell launcher data uses the same seven ids and service names
(`userland/capsule_desktop_shell/src/state/apps.rs:35`). A click first looks up
the service pid. If it exists, desktop shell sends an `NCTL` focus frame to that
pid. If it does not exist, desktop shell sends an `NLAU` launch frame to the init
broker (`userland/capsule_desktop_shell/src/server/handlers/launcher_request/request.rs:19`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_request/request.rs:20`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_request/request.rs:21`,
`userland/capsule_desktop_shell/src/server/handlers/launcher_request/request.rs:23`).

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
+---+---+  +---+----------------+
| NCTL  |  | NLAU to broker     |
| focus |  | allowlisted spawn  |
+-------+  +--------------------+
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

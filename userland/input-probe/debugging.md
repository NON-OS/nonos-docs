# Debugging with input_probe

The probe is a diagnostic, so this page reads two ways: how to tell the probe itself is healthy, and how
to use it to walk the input path back to a break. For what the probe is read the [overview](README.md);
for the receive-and-render internals read [rendering.md](rendering.md); for the path a key travels read
[the input path](../../subsystems/input/path.md).

## The boot marker

When the probe spawns cleanly the kernel prints one line on the boot serial through the shared capsule
boot helper: `[INPUT-PROBE] capsule spawned` (`src/userspace/init/spawn_plan/input_probe_fleet.rs:47`,
`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). That line means the
signature, manifest, attestation, and the `0x1819` capability request all passed and the ELF is mapped.
If spawn fails instead, the helper prints `[ERROR] ` followed by the failure reason
(`src/userspace/init/capsule_boot/run.rs:32`, `src/sys/boot_log/output.rs:49`); the reason names the
stage that rejected the capsule, so a bad key or a mask over the manifest ceiling is legible from serial
alone.

The probe passes an empty `debug_tag`, so there is no per-capsule debug marker beyond the boot line
(`src/userspace/capsule_input_probe/spawn.rs:55`). The tag is only ever printed on an ELF-load error
(`src/kernel_core/process_spawn/capsule_spawn/runner/install/load_elf_into_pid.rs:28`), so seeing nothing
from the tag is the normal, healthy case.

## What a pass looks like

A healthy probe paints a dark slate background and then, as keys arrive, a left-to-right row of white
scaled glyphs, one per printable key, wrapping after 64 characters (`src/setup/mod.rs:51`,
`src/render/mod.rs:19`). Under the inject image the kernel drives a scripted keystream from the timer
tick once the probe has armed its input waiter: three carriage returns, the letters `t e s t`, then three
more carriage returns (`src/interrupts/timer/tick.rs:23`, `src/kernel_core/surface_registry/inject.rs:22`).
The carriage returns are outside the printable range and are filtered, so a passing run shows exactly
`TEST` on glass, upper-cased by the font (`src/render/font.rs:6`, `src/server/runner.rs:38`). If those
four glyphs appear, the whole path is alive: a driver posted into the kernel ring, the router routed a
grabbed delivery over IPC, and the probe decoded it and drew it.

## What a failure looks like, and where it points

The probe fails quietly by design; the loop never faults on a bad message, so a break upstream shows as
an absence, not a crash (`src/server/runner.rs:25`, `src/server/runner.rs:28`). Read the absence by how
far the surface got:

- No surface at all, and no `[INPUT-PROBE] capsule spawned` line. The probe did not spawn. Read the
  `[ERROR]` reason on serial; the mask, signature, or manifest was rejected before the ELF was mapped
  (`src/userspace/init/capsule_boot/run.rs:32`).
- The line printed, but the screen stays blank or unchanged. Bring-up failed and `setup::run` returned
  `Err`, so `_start` exited 2 before the loop ever started (`src/main.rs:22`, `src/setup/mod.rs:18`). The
  cause is a missing `compositor` or `input_router` service, a failed compositor healthcheck, a rejected
  display-info query, or a rejected surface register or share (`src/setup/discover.rs:11`,
  `src/setup/mod.rs:21`, `src/setup/mod.rs:44`). This is a compositor or display problem, not an input
  problem.
- A dark slate surface appears and holds, but no glyphs ever show while you type. Bring-up succeeded and
  the loop is blocked in `mk_ipc_recv_from` with nothing arriving (`src/server/runner.rs:18`). No
  delivery is reaching the probe. The break is upstream: the router refused or dropped the subscribe or
  grab (both are best-effort and their results are discarded, so a refusal is silent,
  `src/server/runner.rs:13`), or nothing is posting into the kernel ring, or the router is not routing to
  this capsule. Confirm the probe is on the router's grab allowlist as `app.input_probe`
  (`userland/capsule_input_router/src/server/handlers/grab_request.rs:25`); a capsule not on that list
  gets no grab and, with the keyboard held elsewhere, may see no keys.
- Glyphs show but the wrong ones, or a fallback box instead of a letter. The path is alive and the bug is
  in resolution or font coverage. The probe does not interpret `flags`, so a shifted or remapped key is
  drawn as whatever `code` the driver and router already resolved (`src/server/runner.rs:39`), and any
  character outside `A`-`Z`, `0`-`9`, and space renders as the fallback box glyph
  (`src/render/font.rs:11`). A wrong letter points at the keymap upstream, not at the probe.

Because the probe holds an exclusive keyboard grab while it runs, it captures the whole keyboard class
ahead of focus, so leaving it up under a normal desktop will make other windows look dead. That is the
grab working, not a bug; it is why the probe is a deliberate test surface (`README.md`,
[the input path](../../subsystems/input/path.md)).

## Source map

```
  src/userspace/init/spawn_plan/input_probe_fleet.rs  the INPUT-PROBE spawn and marker prefix
  src/userspace/init/capsule_boot/run.rs              the spawned and error boot lines
  src/sys/boot_log/output.rs                          the serial line format for ok and error
  src/userspace/capsule_input_probe/spawn.rs          the empty debug_tag and requested caps
  src/main.rs                                          the exit codes for heap and setup failure
  src/setup/mod.rs                                     the bring-up steps that can return Err
  src/setup/discover.rs                               the compositor and router service lookups
  src/server/runner.rs                                the recv loop, best-effort grab, printable filter
  src/render/font.rs                                  the letters, digits, space, and fallback glyph
  src/interrupts/timer/tick.rs                         the inject hook driven from the timer
  src/kernel_core/surface_registry/inject.rs           the scripted TEST keystream
  userland/capsule_input_router/src/server/handlers/grab_request.rs  the grab allowlist
```

Every reference above is verified against those trees.

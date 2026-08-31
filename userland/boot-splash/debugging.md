# Debugging

This page covers the boot marker, the exit codes, and the runtime failure modes. For what each part of the
frame is, see [rendering.md](rendering.md); for the mask and the protocol, see the
[README](README.md).

## Confirm it ran

The first thing to confirm is that the capsule spawned. On a successful boot the kernel prints
`[BOOT-SPLASH] capsule spawned` from the boot path: the tag is `BOOT-SPLASH`, the message is
`capsule spawned` (`src/userspace/init/spawn_plan/desktop_fleet/mod.rs`,
`src/userspace/init/capsule_boot/run.rs:29`, `src/sys/boot_log/output.rs:33`). A signature, manifest, or
capability failure prints an `[ERROR]` line instead (`src/userspace/init/capsule_boot/run.rs:32`,
`src/sys/boot_log/output.rs:49`).

The lower-level spawn also emits `[SPAWN] name=app.boot_splash pid=0x... caps=0x1819 entry=0x...`; the
`caps=0x1819` confirms the admitted mask matches the five bits the splash requests
(`src/kernel_core/process_spawn/capsule_spawn/runner/install/spawn_log.rs:17`, `:18`, `:22`).

## Exit codes

`run` returns a numeric code for each early failure rather than panicking (`src/main.rs:38` through `:62`):

| Code | Meaning | Source |
|------|---------|--------|
| 0 | normal exit after handoff | `src/main.rs:62` |
| 1 | heap init failed | `src/main.rs:32`, `:33` |
| 2 | compositor never came ready after 256 tries | `src/main.rs:41`, `wait_compositor` at `:76` |
| 4 | display-info query failed or returned zero geometry | `src/main.rs:45`, `src/display.rs:43` |
| 5 | surface setup failed (mmap, register, or share) | `src/main.rs:49`, `src/surface.rs:34`, `:48`, `:52` |
| 6 | scene submit was rejected by the compositor | `src/main.rs:56` |

A nonzero exit is a clean, logged failure, not a hang; the splash is written so it can never block the
boot.

## The splash never appears

The capsule waits on `lookup("compositor")` plus a healthcheck up to 256 times, and if the compositor
never registers it returns exit code 2 without painting rather than hanging (`src/main.rs:39`, `:41`,
`wait_compositor` at `:76`, `READY_ATTEMPTS` at `:28`). A missing splash with the compositor absent is the
display core failing to come up, not the splash. If the compositor is up, exit codes 4, 5, and 6 isolate
the next steps: 4 is a bad display query, 5 is a surface setup failure, 6 is a scene-submit rejection
(`src/main.rs:45`, `:49`, `:56`).

## The splash stays on screen and never hands off

Handoff is gated on `desktop_shell` registering plus a one-second settle, with the thirty-second dwell cap
and the eight-million-iteration ceiling as fallbacks (`src/main.rs:103`, `:106`, `:108`; `SETTLE_MS`,
`MAX_DWELL_MS`, `MAX_ITERS` at `:25` through `:27`). A splash that clears quickly to a live desktop means
`desktop_shell` registered on schedule; a splash that hangs on screen for the full dwell before clearing
means the shell never registered and the hard cap fired.

Note that an open detail view intentionally suppresses handoff: the break condition is guarded by
`!show_detail`, so while the `D` detail is open the splash will not exit even past the dwell window
(`src/main.rs:108`, `:116`). A splash that will not leave may simply have the detail view open; press any
key to return and let it hand off.

## The spinner turns but nothing else changes

That is expected. The spinner is a time-based animation driven off elapsed milliseconds (`el / 150`), not
a progress bar, so motion only proves the loop is running, not that the boot is advancing
(`src/main.rs:128`, `src/paint.rs:64`). See [rendering.md](rendering.md#the-status-band).

## The badge reads the wrong verdict

The verdict comes straight from the kernel attestation read (`src/main.rs:52`, `src/paint.rs:56`):

- `verifying` (dim) means `mk_attest_status` returned nonzero, so the badge is `None`, a failed read
  (`src/main.rs:52`, `src/paint.rs:59`). It does not resolve later; the badge is computed once.
- `UNVERIFIED` (amber) is a successful read that reported `zk_verified == 0` (`src/paint.rs:58`). That is a
  kernel or bootloader attestation question, not a splash bug; the real check lives in the
  [proof system](../../subsystems/proof-system/README.md).
- `ATTESTED` (green) is `zk_verified == 1`. Remember the splash only displays this field; a green badge is
  the kernel's claim, covered in the [README security section](README.md#security).

Press `D` to open the detail view and read the actual `kernel blake3` and `zk program hash` values and the
`secure_boot` and `attested` flags behind the verdict (`src/detail.rs:41` through `:47`).

## The D key does nothing

Key delivery depends on the input-router grab. If the router is absent the splash runs without a grab and
never receives keys (`src/main.rs:66`, `:68`). If the grab is denied, the caller is not on the router's
three-name allowlist (`app.boot_splash`, `app.setup_wizard`, `app.input_probe`) and gets `E_ACCES`
(`userland/capsule_input_router/src/server/handlers/grab_request.rs:25`, `:33`). Keys arrive on port 0
with a 50 ms receive timeout, not on the reserved service port (`src/main.rs:112`); a delivery frame
shorter than 40 bytes or with the wrong magic is dropped by `parse_key`
(`src/input.rs:64` through `:70`).

Back to the [README hub](README.md).

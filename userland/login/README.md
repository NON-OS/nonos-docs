# capsule_login

`capsule_login` is the session gate that stands between boot and the desktop. It comes up in the locked
state, paints a lock overlay over the whole screen, and holds the desktop there until a caller asks it to
start a session. Starting a session means unlocking a key in the [keyring](../keyring/README.md); once the keyring
authorizes the unlock, login records the session, tells the desktop shell the session is live, and
repaints to the unlocked color. It is the visible gate and the session bookkeeper. It is not the
credential authority, and it holds no secret of its own. Service `login` on port 4416, reply endpoint
`reply:4417:endpoint.login.reply`, capability mask `0x19`. The source is `userland/capsule_login/`.

## Contents

- [Overview and role](#overview-and-role)
- [Identity](#identity)
- [User reference](#user-reference)
- [Architecture and lifecycle](#architecture-and-lifecycle)
- [Protocol and IPC](#protocol-and-ipc)
- [Security analysis](#security-analysis)
- [How to contribute](#how-to-contribute)
- [Debugging](#debugging)
- [Source map](#source-map)

## Overview and role

Login is a pure IPC service. It has no window, no input subscription, and no line editor; it owns one
full-screen backing surface that it submits to the compositor at z=1 as a lock overlay, and it answers a
four-operation protocol on its service port. Its whole job is a two-state machine: `Locked` at boot, and
`Unlocked` once a session has been started, with the transition gated on the keyring
(`src/state/context/types.rs:28`).

The unlock gate works like this. A caller sends `START_SESSION` with a key id. Login forwards that to the
keyring as `OP_UNLOCK` on behalf of the caller's pid (`src/server/handlers/start_session.rs:20`). If the
keyring authorizes it, login flips its own state to `Unlocked`, notifies the desktop shell that the
session is up, repaints the overlay to the unlocked color, and pings the compositor to present it
(`start_session.rs:24`, `:31`, `:39`, `:40`). Ending a session is the reverse: login relocks its state,
relocks the key through the keyring, notifies the shell, and repaints the locked overlay
(`src/server/handlers/end_session.rs`).

The honest shape of this capsule matters up front. The credential check is not here. Login never sees a
passphrase, a PIN, or any secret bytes; the `START_SESSION` body is a bare 4-byte key id and nothing else
(`src/protocol/limits.rs:4`, `start_session.rs:14`). The thing that decides whether an unlock succeeds is
the [keyring](../keyring/README.md), which owns the key material and applies its own owner-pid check. Login is the
overlay and the session record; the keyring is the lock.

## Identity

Everything the kernel and the service registry need to name and reach login comes from its `Capsule.mk`
and its kernel-side spawn record.

| Field | Value | Source |
|---|---|---|
| Capsule slug | `login` | `Capsule.mk:1` |
| Service handle | `login` | `Capsule.mk:2`, `src/userspace/capsule_login/spawn.rs:29` |
| Namespace | `systems.nonos.login` | `Capsule.mk:7` |
| Service endpoint | `service:4416:login` | `Capsule.mk:8`, `spawn.rs:30` |
| Reply endpoint | `reply:4417:endpoint.login.reply` | `Capsule.mk:9`, `spawn.rs:31`, `:32` |
| Capability mask | `0x19` | `Capsule.mk:11`, `spawn.rs:34` |
| Binary name | `login` | `Capsule.mk:5` |
| Kernel mirror | `src/userspace/capsule_login` | `Capsule.mk:12` |

The mask `0x19` decomposes bit by bit against `src/capabilities/types.rs`:

```
  0x01  CoreExec   bit()  1    types.rs:56
  0x08  IPC        bit()  8    types.rs:59
  0x10  Memory     bit() 16    types.rs:60
  ----
  0x19  = 1 + 8 + 16
```

The kernel spawn path requests exactly `0x19` and no more (`src/userspace/capsule_login/spawn.rs:34`).
Note that the `Capsule.mk` comment reads `IPC | Memory = 0x08 | 0x10 = 0x19`, which is a slip: `0x08 |
0x10` is `0x18`, and the extra bit is `CoreExec` (`0x01`), which every capsule carries to run at all. The
value on the line, `0x19`, is the one that is enforced, and it decodes to CoreExec, IPC, and Memory. There
is no `Crypto` bit (32), no `GraphicsSurfaceCreate` (4096) or `GraphicsPresent` (16384), no `Network` (4),
`FileSystem` (64), or any driver, MMIO, IRQ, DMA, or PIO bit. Login speaks IPC, has a heap, and runs; it
cannot sign, cannot own the screen, cannot touch a device, and cannot reach the network or a filesystem.

## User reference

Login has no keyboard interaction of its own. This is a deliberate and important point: there is no text
field on the lock overlay, no character buffer, and no key handler in the capsule. The overlay is a solid
fill plus a single decorative bar, drawn by `render::paint_locked` and `paint_unlocked`
(`src/render/mod.rs:7`, `:12`); it renders no glyphs and reads no input. Login does not subscribe to input
events at all, and its runner only receives on its service inbox (`src/server/runner.rs:30`).

What a person sees, and what actually drives the state, are two different layers:

| What the user perceives | What the code does | Source |
|---|---|---|
| A locked screen at boot | `Context` starts in `SessionState::Locked`; the overlay is painted and submitted at z=1 | `src/state/context/new.rs:37`, `src/setup/run.rs:37`, `:45` |
| The screen unlocks | some caller sent `START_SESSION`; the keyring authorized the unlock and login repainted to the unlocked color | `start_session.rs:20`, `:39` |
| The screen relocks | some caller sent `END_SESSION`; login relocked and repainted the locked overlay | `end_session.rs:11`, `:22` |

The passphrase prompt a user expects to type into is not part of this capsule. If there is a credential
UI in a given desktop profile, it lives in the caller that decides which key id to unlock and sends
`START_SESSION`; login only receives the resulting key id and asks the keyring. So the four "actions" that
change what the user sees are the four protocol operations below, each driven by a caller over IPC rather
than by a keystroke into login.

## Architecture and lifecycle

The capsule is `no_std`/`no_main`. `_start` initializes the heap, blocks in `wait_for_setup` until setup
succeeds, then enters the server loop (`src/main.rs:32`). The top-level modules are `clients` (the outbound
IPC to keyring, desktop shell, and compositor), `protocol` (the wire format and ops), `render` (the
overlay painter), `server` (the receive loop and handlers), `setup` (peer discovery and surface bring-up),
and `state` (the session machine) (`src/main.rs:21`).

Startup and setup (`src/setup/run.rs`):

1. `_start` calls `wait_for_setup`, which retries `setup::run` in a yield loop until it returns a
   `Context`, so login parks harmlessly if its peers are not up yet (`src/wait_for_setup.rs:18`).
2. `setup::run` discovers three peers by name through `mk_service_lookup`: the keyring, the desktop shell,
   and the compositor (`src/setup/run.rs:22`, `src/setup/discover/lookup_port.rs:19`). A zero port or a
   zero pid is a failed lookup and setup restarts.
3. It health-checks the compositor, queries the display dimensions, and allocates a private backing
   surface sized to the screen (`src/setup/run.rs:25`, `:26`, `:27`).
4. It paints the locked overlay into that surface, registers and shares the surface with the compositor,
   and submits it as a scene layer at `OVERLAY_Z = 1` (`src/setup/run.rs:37`, `:45`,
   `src/setup/register.rs:21`, `src/setup/constants.rs:18`). If the register or the scene submit fails, it
   tears down the surface and backing and retries, so a half-registered overlay is never left behind
   (`src/setup/run.rs:38`, `:57`).

The render path is trivial and self-contained: `fill` writes a solid ARGB background across the surface
and `paint_bar` draws one horizontal bar, both with volatile writes (`src/render/mod.rs:35`, `:17`).
Locked is `0xFF242A36` with the bar near the top, unlocked is `0xFF143A22` with the bar lower
(`src/render/mod.rs:3`, `:4`). There is no font, no cursor, and no text.

The input path is IPC, not keyboard. The server loop receives on service inbox 0, blocking for the first
message of a batch and then draining without waiting until the inbox is empty (`src/server/runner.rs:24`,
`:28`). Each message is parsed against the 20-byte header (magic, version, op, flags, request id, payload
length) and dispatched by op; a malformed frame gets the parser's errno straight back
(`src/protocol/decode.rs:3`, `runner.rs:35`). A request that carries a body on an op that expects an empty
body is rejected with `E_INVAL`, and an unknown op with an empty body is `E_BAD_OP` (`runner.rs:51`, `:54`,
`:52`).

The session state is a two-variant enum on the `Context` (`src/state/context/types.rs:28`).
`start_session` refuses a second concurrent unlock with `E_BUSY` and bumps a wrapping 32-bit serial each
time it opens a session (`src/state/context/start_session.rs:21`, `:25`). `end_session` is owner-checked:
it returns `E_AUTH` if the caller is not the pid that opened the session, and it is a no-op that returns
`Ok` if the state is already locked (`src/state/context/end_session.rs:22`, `:24`). `get_state` projects
the machine to three words, `(state, owner_pid, serial)`, where `Locked` is all zeros
(`src/state/context/state_words.rs:19`).

## Protocol and IPC

The frame is a 20-byte header, magic `NLGN` (`0x4E4C474E`), version 1, then op, flags, request id, and an
explicit payload length; the payload follows (`src/protocol/header.rs:1`, `src/protocol/decode.rs:36`).
Four operations (`src/protocol/ops.rs`):

```
  OP_HEALTHCHECK    0x0001    empty body -> status
  OP_START_SESSION  0x0002    body = key_id:u32 (4 bytes) -> status
  OP_END_SESSION    0x0003    empty body -> status
  OP_GET_STATE      0x0004    empty body -> status + state:u32 owner_pid:u32 serial:u32
```

`START_SESSION` is the core. Its body is exactly 4 bytes, a little-endian key id; any other length is
`E_INVAL` (`src/protocol/limits.rs:4`, `src/server/handlers/start_session.rs:10`). The handler runs, in
order (`start_session.rs`):

```
  start_session(req, sender_pid):
      key_id = body[0..4]                                                   // else E_INVAL
      keyring::unlock(keyring_port, req.request_id, sender_pid, key_id)     // OP_UNLOCK; else pass errno through
      serial = ctx.start_session(sender_pid, key_id)                        // Locked -> Unlocked; else E_BUSY
      desktop_shell::notify_info(shell_port, req.request_id ^ serial, "login:session_unlocked")
      render::paint_unlocked(ctx)
      compositor::ping_damage(compositor_port, req.request_id ^ serial)     // present the repaint
      status 0
```

The keyring call is the credential gate. Login's keyring client sends `OP_UNLOCK`, op number `5`, with the
caller pid and the key id, over an 8-byte header plus an 8-byte body (`src/clients/keyring.rs:9`, `:27`,
`:11`). This is the same `UNLOCK` op the [keyring](../keyring/README.md) documents as op 5; the matching relock is
`OP_LOCK`, op `4` (`src/clients/keyring.rs:8`, `:36`). If the keyring returns a nonzero status, login
returns that status verbatim, so a login failure that is really a keyring rejection carries the keyring's
errno, not a login-specific one (`start_session.rs:20`).

The signal that launches the desktop is the desktop-shell notify. After the unlock and the state flip,
login sends the shell an `OP_NOTIFY` (op `5`) info message with the fixed text `login:session_unlocked`
(`src/server/handlers/start_session.rs:7`, `:31`, `src/clients/desktop_shell.rs:9`, `:13`). That is the
handoff that tells the shell the gate is open. `END_SESSION` sends the mirror message
`login:session_locked` (`src/server/handlers/end_session.rs:7`, `:18`).

`start_session` is transactional. If either follow-on call fails, the notify or the compositor damage
ping, login rolls the whole thing back: it ends the session it just started, relocks the key through the
keyring, and returns `E_NOTREADY`, so a session is never left open with a dead overlay
(`start_session.rs:31`, `:40`). Each of these outbound calls carries a request id of `req.request_id ^
serial`, which the shell can correlate back to the session by xor.

The three peer clients each speak their own wire. The keyring client is an 8-byte header, no magic
(`src/clients/keyring.rs:6`). The desktop-shell client uses magic `NDSH` (`0x4E445348`), version 1, op
`OP_NOTIFY` `0x0005` (`src/clients/desktop_shell.rs:6`, `:9`). The compositor client uses magic `NCMP`
(`0x4E434D50`), version 1, with `OP_HEALTHCHECK 0x0001`, `OP_SCENE_SUBMIT 0x0002`, and `OP_DAMAGE_COMMIT
0x0003` (`src/clients/compositor/constants.rs:16`, `:19`).

## Security analysis

The interesting security property of login is what it does not hold and does not see. It is a
trust-sensitive capsule by position, since it gates the whole desktop, but it is deliberately thin.

- Login is not the credential authority, and there is no passphrase in it to protect. The `START_SESSION`
  body is a 4-byte key id; there are no secret bytes in the request, no passphrase buffer in the `Context`,
  and therefore nothing here to zeroize (`src/protocol/limits.rs:4`, `src/state/context/types.rs:16`). The
  question "is the passphrase zeroized" does not apply to this capsule, because the passphrase never
  reaches it. The actual key material and the actual verification live in the [keyring](../keyring/README.md), which
  owns the secret, applies its own owner-pid check on `UNLOCK`, and follows its own wiping discipline. A
  compromise of login cannot forge an unlock, because login never held the key or the check; the worst it
  can do is fail to gate, and the keyring still refuses an unlock it does not authorize.

- The unlock authorizes exactly one thing: it moves login's own state to `Unlocked` for one owner pid and
  one key id, and it flips the keyring's lock flag on that key so the key can be used
  (`src/state/context/start_session.rs:27`, `src/clients/keyring.rs:27`). It does not hand any key material
  to login. What an unlocked session actually unlocks downstream is the keyring's decision, not login's.

- The capability mask is the least-privilege argument. `0x19` is CoreExec, IPC, and Memory only
  (`Capsule.mk:11`, `spawn.rs:34`). Notably login does not hold `GraphicsSurfaceCreate` or
  `GraphicsPresent` the way the compositor does; it registers a backing surface through the shared surface
  path during setup and submits it at z=1, but it does not own the screen (`src/setup/register.rs:21`,
  `src/setup/run.rs:45`). It holds no `Crypto`, which is the whole point: the capsule that gates the
  desktop cannot itself sign or hold a key. It holds no `Network` and no `FileSystem`, so it cannot ship a
  session state off the machine or persist one to storage.

- Session ownership is pid-bound. The `Unlocked` state records the `owner_pid` that opened it, and
  `end_session` returns `E_AUTH` to any other caller, so a second capsule cannot close or hijack the first
  one's session over the port (`src/state/context/end_session.rs:24`).

- The honest boundary: the gate is a policy overlay, not a mandatory access control. Login paints a lock
  overlay and tracks session state, but a capsule that never asks login is not blocked by login. The real
  enforcement of what an unlocked session grants is the keyring's, which only makes a key usable after its
  own `UNLOCK`. There is also no session timeout, no rate limiting on start attempts, and the serial is a
  wrapping 32-bit counter, so login is the session bookkeeper and the visible gate, not a second factor.

## How to contribute

The source lives at `userland/capsule_login/`. The wire protocol is under `src/protocol/`, the receive
loop and the four handlers under `src/server/`, the outbound peer clients under `src/clients/`, the
setup and overlay bring-up under `src/setup/`, the overlay painter in `src/render/mod.rs`, and the session
state machine under `src/state/context/`.

To change a handler, edit the one file for that op under `src/server/handlers/` (`start_session.rs`,
`end_session.rs`, `get_state.rs`, `health.rs`) and, if you touch the wire, keep `src/protocol/ops.rs`,
`src/protocol/limits.rs`, and `src/protocol/decode.rs` in step; the runner dispatches on the op constants
in `src/server/runner.rs:42`. To change what login says to a peer, edit the matching client under
`src/clients/`; the keyring op numbers in particular (`OP_UNLOCK 5`, `OP_LOCK 4`) must match the
[keyring](../keyring/README.md) side (`src/clients/keyring.rs:8`).

To build and sign the capsule, use the generated per-slug make targets (`nonos-mk/capsule.mk:158`,
`:182`, `:261`, `:263`, `:184`), included through `userland/capsule_login/Capsule.mk:14`:

```
  make nonos-mk-login              build the capsule ELF
  make nonos-mk-login-sign         produce the id cert, manifest, and attestation trailer
  make nonos-mk-login-verify       verify the signed artifacts against the trust anchor
  make nonos-mk-check-login-keys   check the per-capsule signing keys exist
```

Login is part of the desktop fleet, so a full desktop image that includes it is built by the desktop-gui
and full-gui profiles, which pull in `$(login_ARTIFACTS)` (`Makefile:1087`, `Makefile:1096`).

Code standards the capsule must meet: `cargo fmt` and a clean `cargo clippy`; no panics, `unwrap`, or
`expect` in capsule code (every handler returns errors as a status word, never a panic; the peer clients
map a short read to an errno rather than unwrapping, `src/clients/keyring.rs:24`); modular files, one unit
per file, with `mod.rs` used only for re-exports (`src/setup/mod.rs`, `src/protocol/mod.rs`); and the AGPL
header at the top of every source file, matching the header already on every module (`src/main.rs:1`).

## Debugging

The first thing to confirm is that login came up. It is spawned in the desktop-services fleet as
`boot::capsule("LOGIN", "login", ...)` (`src/userspace/init/spawn_plan/desktop_services.rs:66`), which on
success prints `[LOGIN] capsule spawned` through `boot_log::ok` (`src/userspace/init/capsule_boot/run.rs:29`,
`src/sys/boot_log/output.rs:33`). An absent line means the capsule never started, usually a signature,
manifest, or capability failure, and the error path prints an `[ERROR]` line instead
(`src/userspace/init/capsule_boot/run.rs:32`). Login is also one of the names the spawn tracer emits
install-stage lines for, so a stall during its install is visible on the console
(`src/kernel_core/process_spawn/capsule_spawn/runner/install/trace.rs:20`).

Setup depends on ordering. Login must find three peers by name before it can serve: the keyring, the
desktop shell, and the compositor (`src/setup/run.rs:22`). If any lookup returns a zero port or pid,
`lookup_port` returns `"service lookup failed"` and `wait_for_setup` retries in a yield loop rather than
finishing (`src/setup/discover/lookup_port.rs:24`, `src/wait_for_setup.rs:18`). So a login that never
paints its overlay is usually a peer that has not registered yet: the keyring and compositor must come up
first.

The runtime failure signatures are specific:

- Desktop never unlocks after a `START_SESSION`. If the keyring refused the unlock, login returns the
  keyring's own errno unchanged (`start_session.rs:20`); check the keyring side, since an `EACCES` there
  is the keyring's owner-pid check on the key. If the unlock succeeded but the shell notify or the
  compositor damage ping failed, login rolls back and returns `E_NOTREADY` after relocking, which is the
  signature of the desktop coming up out of order (`start_session.rs:31`, `:40`).
- A `START_SESSION` with the wrong body length is `E_INVAL`; only a 4-byte body is accepted
  (`start_session.rs:10`).
- A second `START_SESSION` while a session is already open is `E_BUSY` (`src/state/context/start_session.rs:22`).
- An `END_SESSION` from a pid that did not open the session is `E_AUTH`
  (`src/state/context/end_session.rs:24`).
- A malformed frame is rejected by the parser before dispatch: `E_BAD_MAGIC` for a wrong magic,
  `E_BAD_VERSION` for a wrong version, `E_BAD_LEN` for a truncated or mismatched-length frame, and
  `E_BAD_OP` for an unknown op with an empty body (`src/protocol/decode.rs:23`, `:29`, `:36`,
  `src/server/runner.rs:52`).

Login holds no `Debug` capability and emits no diagnostic output of its own at runtime; the boot marker and
the caller-side status words are the debugging surface.

## Source map

```
  userland/capsule_login/src/main.rs                        _start -> wait_for_setup -> server::run
  userland/capsule_login/src/setup/run.rs                   peer discovery, overlay paint, submit at z=1
  userland/capsule_login/src/setup/discover/lookup_port.rs  the mk_service_lookup wrapper, "service lookup failed"
  userland/capsule_login/src/setup/register.rs              register + share the backing surface
  userland/capsule_login/src/render/mod.rs                  the locked/unlocked overlay painter (no text)
  userland/capsule_login/src/server/runner.rs               the receive loop and op dispatch
  userland/capsule_login/src/server/handlers/start_session.rs   unlock + notify + rollback, E_INVAL/E_NOTREADY
  userland/capsule_login/src/server/handlers/end_session.rs     relock + notify, E_AUTH
  userland/capsule_login/src/server/handlers/get_state.rs       state projection
  userland/capsule_login/src/clients/keyring.rs             OP_UNLOCK (5) / OP_LOCK (4) calls
  userland/capsule_login/src/clients/desktop_shell.rs       OP_NOTIFY (5), "login:session_unlocked"
  userland/capsule_login/src/clients/compositor/            healthcheck, scene submit, damage ping
  userland/capsule_login/src/protocol/                      NLGN header, ops, limits, errnos, decode
  userland/capsule_login/src/state/context/                 the Locked/Unlocked state machine
  userland/capsule_login/Capsule.mk                         slug, endpoints, CAPSULE_REQUIRED_CAPS = 0x19
  src/userspace/capsule_login/spawn.rs                      the kernel-side embed and verified spawn
  src/userspace/init/spawn_plan/desktop_services.rs         the desktop-fleet spawn entry (LOGIN)
  nonos-mk/capsule.mk                                       the generated nonos-mk-login[-sign|-verify] targets
```

Every reference above is verified against those trees.
</content>
</invoke>

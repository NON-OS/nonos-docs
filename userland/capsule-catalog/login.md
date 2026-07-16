# capsule_login

The login capsule gates the desktop behind a session: it paints a lock overlay, starts a session by
unlocking the keyring, and ends it by re-locking. It does not verify credentials itself; the keyring is
the authority. Service `login` on port 4416, capability mask `0x19`. The source is
`userland/capsule_login/`.

## The server loop

`main.rs:32` waits for setup (discovering the keyring, desktop shell, and compositor, allocating a
backing surface, and painting the locked overlay at z=1), then runs the drain loop
(`src/server/runner.rs`). The frame is `NLGN` (magic `0x4E4C474E`), version 1.

## The operations

Four operations (`src/protocol/ops.rs`):

```
  HEALTHCHECK=1  START_SESSION=2  END_SESSION=3  GET_STATE=4
```

`START_SESSION` (`src/server/handlers/start_session.rs:9`) is the core:

```
  start_session(req, sender_pid):
      key_id = body
      keyring::unlock(keyring_port, sender_pid, key_id)      // OP_UNLOCK -> the keyring verifies
      ctx.start_session(sender_pid, key_id)                   // Locked -> Unlocked{owner, serial}
      desktop_shell::notify("login:session_unlocked")
      repaint to the unlocked color; damage the compositor
```

The credential check lives in the [keyring](keyring.md): login sends `OP_UNLOCK` with the caller pid and
the key id, and the keyring is authoritative on whether it succeeds. Login only tracks the resulting
session state. `END_SESSION` re-locks the key and returns to the locked overlay.

## State and honesty

The `Context` (`src/state/context/types.rs:16`) holds the peer ports, the backing surface, a serial, and
a `SessionState` that is either `Locked` or `Unlocked { owner_pid, key_id, serial }`. The owner is
enforced: `end_session` (`state/context/end_session.rs:24`) returns `E_AUTH` if the caller is not the pid
that started the session, so one capsule cannot end another's session. Honest gaps: there is no session
timeout (the session stays unlocked until an explicit end), no rate limiting on start attempts, and the
serial is a wrapping 32-bit counter that the shell correlates by xor with the request id.

`start_session` is transactional. After the keyring unlock succeeds it starts the session, then notifies
the shell and pings the compositor for damage (`src/server/handlers/start_session.rs`). If either of
those follow-on calls fails it rolls the whole thing back: it ends the session it just started and
re-locks the key through the keyring, then returns `E_NOTREADY`, so a half-opened session does not leave
the key unlocked with no live overlay.

## Security analysis

The mask is `0x19` (`CAPSULE_REQUIRED_CAPS` in `userland/capsule_login/Capsule.mk`), decoding to
`CoreExec | IPC | Memory` against `src/capabilities/types.rs`. That is deliberately thin for a capsule
that gates the whole desktop. Notably it does not hold the graphics-create or present capabilities the way
the compositor does: it registers a backing surface through the shared surface path during setup
(`src/setup/register.rs:37`) and submits it to the compositor at z=1, but it does not itself own the
screen. It holds no crypto capability either, which is the whole point of the design.

- **Login is not the credential authority.** The mask has no `Crypto` and login stores no secret. The
  actual verification lives in the [keyring](keyring.md): login sends `OP_UNLOCK` (op 5) with the caller
  pid and key id (`src/clients/keyring.rs:27`) and trusts the keyring's status. A compromise of login
  cannot forge an unlock, because it never held the key material or the check; the worst it can do is fail
  to gate, and the keyring still refuses an unlock it does not authorize.
- **Session ownership is pid-bound.** The `Unlocked` state records the `owner_pid` that opened it, and
  `end_session` returns `E_AUTH` to any other caller, so a second capsule cannot close or hijack the first
  one's session over the port.
- **Honest boundary: the gate is a policy overlay, not a mandatory access control.** Login paints a lock
  overlay and tracks session state, but a capsule that never asks login is not blocked by login; the real
  enforcement of what an unlocked session unlocks is the keyring's, which only hands out key material after
  its own unlock. Login is the session bookkeeper and the visible gate, and the keyring is the lock.

## Debugging

Login registers as `login` on port 4416 and is reached by `mk_service_lookup("login")`. During setup it
must find three peers by name (`src/setup/discover/`): the keyring, the desktop shell, and the compositor.
If any lookup returns a zero port or pid, `lookup_port` returns `"service lookup failed"`
(`src/setup/discover/lookup_port.rs`) and login does not finish setup, so the ordering dependency is that
the keyring and compositor come up first. The kernel spawn marker is:

```
  [SPAWN] name=login pid=0x... caps=0x19 entry=0x...
```

`caps=0x19` confirms it was admitted with `CoreExec | IPC | Memory` only. `login` is one of the names the
spawn tracer emits install-stage lines for (`src/kernel_core/process_spawn/capsule_spawn/runner/install/trace.rs:20`),
so a stall during its install is visible on the console.

The runtime failure signatures are specific. A `START_SESSION` with the wrong body length is `E_INVAL`. An
unlock the keyring refuses comes back as the keyring's own errno passed straight through
(`src/server/handlers/start_session.rs`), so a login failure that is really a keyring rejection carries the
keyring's status, not a login-specific one. A `START_SESSION` that unlocks but then cannot reach the shell
or compositor is `E_NOTREADY` after the rollback, which is the signature of the desktop coming up out of
order. An `END_SESSION` from a pid that did not open the session is `E_AUTH`.

## Source map

```
  userland/capsule_login/src/setup/run.rs                  peer discovery, overlay paint at z=1
  userland/capsule_login/src/setup/discover/lookup_port.rs the mk_service_lookup wrapper, "service lookup failed"
  userland/capsule_login/src/server/handlers/start_session.rs, end_session.rs   unlock + rollback, E_AUTH
  userland/capsule_login/src/clients/keyring.rs            OP_UNLOCK (5) / OP_LOCK (4) calls
  userland/capsule_login/src/state/context/                the Locked/Unlocked state machine
  userland/capsule_login/Capsule.mk                        CAPSULE_REQUIRED_CAPS = 0x19, endpoint 4416
```

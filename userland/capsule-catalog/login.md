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
enforced: `end_session` (`state/context/end_session.rs`) returns `E_AUTH` if the caller is not the pid
that started the session, so one capsule cannot end another's session. Honest gaps: there is no session
timeout (the session stays unlocked until an explicit end), no rate limiting on start attempts, and the
serial is a wrapping 32-bit counter that the shell correlates by xor with the request id.

## Source

```
  userland/capsule_login/src/setup/run.rs                   peer discovery, overlay paint
  userland/capsule_login/src/server/handlers/start_session.rs, end_session.rs
  userland/capsule_login/src/clients/keyring.rs             the OP_UNLOCK / OP_LOCK calls
  userland/capsule_login/src/state/context/                 the session state machine
```

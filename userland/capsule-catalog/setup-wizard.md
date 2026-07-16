# capsule_setup_wizard

The setup wizard is the first-run configuration flow: a sequence of full-screen steps that collect the
language, keyboard, keys, passphrase, persistence, network, admin, privacy, and appearance choices, and
commit them to policy. App `app.setup_wizard` on port 4794, capability mask `0x1819`, a direct GUI
runner. The source is `userland/capsule_setup_wizard/`.

## The flow

`main.rs:15` initializes the heap, runs setup (discovering the compositor, input router, and policy,
allocating a surface, submitting it to the compositor), and runs the loop (`src/server/runner.rs:12`):

```
  run(ctx):
      input_router::subscribe(); input_router::grab_keyboard()
      redraw(ctx)
      loop:
          ev = recv keyboard event
          outcome = screens::on_key(ctx, ev.code)
          ctx.step = apply(ctx.step, outcome)      // advance / go back
          if ctx.step >= DONE:  mk_exit(0)
          redraw(ctx)
```

It grabs the keyboard (it is one of the three trusted grabbers the [input router](input-router.md)
allows) and drives a ten-screen state machine on key presses. Enter advances, Escape goes back, and
numeric and `j`/`k` keys navigate lists.

## The screens

`screens::draw` (`src/render/screens/mod.rs:15`) selects the screen by `ctx.step`: language, keyboard,
keygen, passphrase, persistence, network, admin, privacy, appearance, and review. The review screen
(`src/render/screens/review.rs:21`) commits every choice to the [policy](policy.md) service:

```
  commit(ctx):
      policy::set_u8(Language, ...); set_u8(KeyboardLayout, ...); set_i8(Timezone, ...)
      set_u8(Wallpaper, ...); set_u8(Theme, ...)
      set_bool(AnonymousMode, ...); set_bool(WifiAutoconnect, ...)
      set_bool(AutoWipe, ...); set_bool(NymEnabled, ...)
      set_str(Hostname, ...)
```

The wizard is one of the two capsules policy trusts to write (the other is the settings app), so its
commits take effect.

## State and honesty

The `Context` (`src/state.rs`) holds the current step and every selection, plus buffers for the
passphrase, admin name, and hostname. Honest gaps stated from the code: the `keygen` screen advances a UI
stage but does not itself generate keys (the actual key generation is deferred), the passphrase entry has
no masking or strength check, the `persist_sel` choice is recorded in policy but the wizard does not
itself configure or encrypt a persistent store, policy is optional (if its port is absent the wizard
still completes but the settings are lost), and there is no abort-and-reboot from the final review.

## Security analysis

The mask is `0x1819` (`CAPSULE_REQUIRED_CAPS` in `userland/capsule_setup_wizard/Capsule.mk`), decoding to
`CoreExec | IPC | Memory | GraphicsDisplayQuery | GraphicsSurfaceCreate` against `src/capabilities/types.rs`.
It is the same graphics-client mask as the other full-screen capsules: it queries the display, registers one
surface (`src/setup/mod.rs:44`), submits it to the compositor, and drives it from keyboard events. It holds
no `GraphicsPresent`, and no crypto, hardware, filesystem, or network capability.

- **The authority here is policy write, not the caps.** The wizard's power is not in its capability mask,
  which is ordinary; it is that the [policy](policy.md) service trusts this capsule by name to write. The
  review screen (`src/render/screens/review.rs:21`) commits every choice with `policy::set_*` and those
  writes take effect because the wizard is one of the two names policy allows to write (the other is the
  settings app). So the boundary that matters is on policy's side, not the wizard's: policy is what decides
  the wizard may write, and the wizard cannot grant itself any other authority through that channel.
- **Trusted keyboard grabber.** The wizard is one of the three names the [input router](input-router.md)
  allows to take an exclusive grab (as `app.setup_wizard`), and it grabs the keyboard so first-run entry
  cannot be observed by a background subscriber. That grant is the router's, gated by name.
- **Honest boundary: the wizard records intent, it does not enact secrets.** As the state note says, the
  keygen screen advances a UI stage without generating keys, the passphrase entry has no masking or strength
  check, and the persistence choice is recorded in policy without the wizard configuring or encrypting a
  store. So the security-sensitive parts of first-run (key material, an encrypted persistent store) are
  deferred to other capsules; the wizard collects and commits the configuration, it does not itself hold or
  create the secrets those choices imply.

## Debugging

The wizard runs as `app.setup_wizard` on port 4794. Its setup discovers the compositor, input router, and
policy by name; the input router and compositor must be up for it to grab the keyboard and paint, while
policy is optional (if its port is absent the wizard still completes but the settings are lost, per the
honesty note). The kernel spawn marker is:

```
  [SPAWN] name=app.setup_wizard pid=0x... caps=0x1819 entry=0x...
```

`caps=0x1819` confirms the admitted mask. The clearest live signal is the grab: the wizard calls
`input_router::grab_keyboard` at the top of its loop (`src/server/runner.rs`), and if the router refuses
(the wizard is not resolved as a trusted grabber, or the keyboard is already held) that grab comes back
`E_ACCES` or `E_BUSY` from the [input router](input-router.md) and the wizard receives no keys. A wizard on
screen that does not respond to Enter or Escape is that grab failing, not the state machine wedging. The
failure signature at the end is clean exit: reaching the review and advancing past `DONE` calls `mk_exit(0)`
(`src/server/runner.rs`), so a wizard that completes disappears rather than lingering, and its policy commits
are visible as the settings other capsules then read.

## Source map

```
  userland/capsule_setup_wizard/src/server/runner.rs           the key-driven loop, grab_keyboard, mk_exit
  userland/capsule_setup_wizard/src/setup/mod.rs               surface register + compositor submit
  userland/capsule_setup_wizard/src/render/screens/            the ten screens
  userland/capsule_setup_wizard/src/render/screens/review.rs   the policy commit (the trusted write)
  userland/capsule_setup_wizard/src/state.rs                   the collected configuration
  userland/capsule_setup_wizard/Capsule.mk                     CAPSULE_REQUIRED_CAPS = 0x1819, endpoint 4794
```

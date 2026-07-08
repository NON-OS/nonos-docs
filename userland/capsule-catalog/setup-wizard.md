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

## Source

```
  userland/capsule_setup_wizard/src/server/runner.rs      the key-driven loop
  userland/capsule_setup_wizard/src/render/screens/       the ten screens
  userland/capsule_setup_wizard/src/render/screens/review.rs   the policy commit
  userland/capsule_setup_wizard/src/state.rs              the collected configuration
```

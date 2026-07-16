# capsule_boot_splash

The boot splash is the screen shown while the desktop comes up, and it carries the attestation badge. It
registers no service; it is a client that renders a splash, reads the kernel's attestation status, and
hands the screen to the desktop once the shell is up. The source is `userland/capsule_boot_splash/`.

## The client flow

`main.rs:31` initializes the heap and runs a linear sequence (`main.rs:33`):

```
  run():
      comp = wait_compositor()                  // poll lookup("compositor") up to 256 times
      (w, h, stride) = display::query(comp)
      (base, handle) = surface::setup(w, h)
      badge = mk_attest_status(&att) == 0 ? Some(att.zk_verified == 1) : None
      paint::splash(base, w, h, badge)
      scene::submit(comp, handle); scene::damage(comp)
      interact(...)                             // wait for desktop_shell, handle the D key
      scene::remove(comp); mk_surface_release(handle)
```

It talks to the compositor with the same `NCMP` protocol other clients use (healthcheck, display query,
scene submit, damage, remove).

## The attestation badge

The splash's distinctive job is the attestation badge (`main.rs:51`). It reads the kernel's attestation
status with `mk_attest_status`, which returns a struct with a `zk_verified` flag, a `secure_boot` flag,
the kernel BLAKE3 hash, and the ZK program hash. The badge is green when `zk_verified` is set. Pressing
`D` (`src/detail.rs`) shows the detail view: the kernel hash and the ZK program hash in hex, with the
secure-boot and zk-verified flags colored by state.

The honest framing matters here as it did for [attest](attest.md): the boot splash **displays** the
kernel's attestation, it does not verify a proof itself. The badge's trust is the kernel's attestation
field, so it reflects what the kernel measured, and the real
[verification](../../subsystems/proof-system/README.md) is the kernel's and the bootloader's.

## Handoff

`interact` (`main.rs:88`) polls for the `desktop_shell` service; once it registers and a one-second
settle passes, or a maximum dwell of thirty seconds elapses, the splash removes its scene and exits,
handing the screen to the desktop. It animates a spinner in the meantime (a time-based cycle, not a
progress metric). If the compositor dies mid-splash it exits rather than recovering.

## Security analysis

The mask is `0x1819` (`CAPSULE_REQUIRED_CAPS` in `userland/capsule_boot_splash/Capsule.mk`), decoding to
`CoreExec | IPC | Memory | GraphicsDisplayQuery | GraphicsSurfaceCreate` against `src/capabilities/types.rs`.
That is the same graphics-client mask the [wallpaper](wallpaper.md) and [login](login.md) capsules carry: it
queries the display, registers one surface (`src/surface.rs:47`), submits it to the compositor, and reads
attestation. It holds no `GraphicsPresent`, so it does not touch the framebuffer directly, and it holds no
crypto, hardware, filesystem, or network capability.

- **It displays, it does not verify.** This is the security-relevant point and it is worth restating from
  the body. The badge is `mk_attest_status`'s `zk_verified` field, read out of the kernel
  (`src/main.rs:51`). The splash has no `Crypto` capability and runs no verifier; it renders whatever flags
  the kernel measured. The trust in that green badge is the kernel's and the bootloader's attestation, not
  the splash's, so a compromised splash could paint a green badge on an unverified system but could not make
  the system actually verified, and equally could not weaken a real verification, because it is downstream
  of it. The [attest](attest.md) page and the [proof system](../../subsystems/proof-system/README.md) carry
  the real check.
- **Almost no reach.** Beyond drawing one surface and reading a status word, the splash cannot do anything.
  It is one of the three names the [input router](input-router.md) trusts to take an exclusive grab (as
  `app.boot_splash`), which is how it reads the `D` key, but that grant is the router's, gated by name, and
  the splash gives up the screen and exits when the shell comes up.

## Debugging

The boot splash registers no service of its own. `Capsule.mk` reserves the endpoint
`service:4796:app.boot_splash`, but the code never binds or receives on 4796; its input loop calls
`mk_ipc_recv_from(0, ...)` on port 0 (`src/main.rs:112`) to poll for keys and the shell, so there is no
`lookup("app.boot_splash")` to fail against. It is a pure client. The kernel spawn marker is:

```
  [SPAWN] name=app.boot_splash pid=0x... caps=0x1819 entry=0x...
```

`caps=0x1819` confirms the admitted mask. The way you tell the splash is alive is the screen: it polls
`lookup("compositor")` up to 256 times (`main.rs`), and if the compositor never registers the splash never
paints and exits rather than hanging. The failure signature at the other end is the handoff: `interact`
(`main.rs:88`) waits for `desktop_shell` to register plus a one-second settle, or a thirty-second maximum
dwell, then removes its scene and exits. A splash that stays on screen past that maximum dwell means the
compositor died mid-splash (it exits rather than recovering); a splash that clears quickly to a live desktop
means `desktop_shell` registered on schedule. The spinner is a time-based animation, not a progress bar, so
a spinning splash is not evidence of forward progress.

## Source map

```
  userland/capsule_boot_splash/src/main.rs        the client flow, attestation read (main.rs:51), handoff (main.rs:88)
  userland/capsule_boot_splash/src/surface.rs     the single registered splash surface
  userland/capsule_boot_splash/src/paint.rs       the splash render and badge
  userland/capsule_boot_splash/src/detail.rs      the D-key detail view (kernel + ZK hashes)
  userland/capsule_boot_splash/Capsule.mk         CAPSULE_REQUIRED_CAPS = 0x1819 (endpoint reserved, unused)
```

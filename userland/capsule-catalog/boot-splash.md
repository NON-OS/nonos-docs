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

## Source

```
  userland/capsule_boot_splash/src/main.rs       the client flow, attestation read, handoff
  userland/capsule_boot_splash/src/paint.rs       the splash render
  userland/capsule_boot_splash/src/detail.rs      the D-key detail view (kernel + ZK hashes)
```

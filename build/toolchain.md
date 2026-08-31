# Toolchain

This page describes the x86_64 user target, cargo build flags, Make targets,
and how capsules compile. Read [Build](README.md), then
[Signing](signing.md).

---

## 1. Rust targets

The userland target is `userland/x86_64-nonos-user.json`. It uses
`x86_64-unknown-none-elf`, target arch `x86_64`, vendor `nonos`, OS `none`, and
64-bit little-endian pointers (`userland/x86_64-nonos-user.json:2`). It sets
`panic-strategy` to `abort`, uses `rust-lld`, links with `-nostdlib`, `-pie`,
`--gc-sections`, and 4 KiB max page size, then sets PIC and static PIE output
(`userland/x86_64-nonos-user.json:30`).

The kernel target is `x86_64-nonos.json`. It uses the same LLVM target but
disables the red zone (`x86_64-nonos.json:16`), links `-no-pie`
(`x86_64-nonos.json:27`), sets relocation model `static` (`:30`), and uses code
model `kernel` (`:31`).

| Target file | Intended binary | Link posture |
|-------------|-----------------|--------------|
| `userland/x86_64-nonos-user.json` | Capsule ELF | PIE, PIC, no default libraries |
| `x86_64-nonos.json` | Kernel ELF | static, kernel code model, no default libraries |

## 2. Toolchain bootstrap

The build pins Rust to `nightly-2026-01-16` (`mk/00-config.mk:55`), matching
`rust-toolchain.toml`. The toolchain stamp rule installs the pinned toolchain,
adds `x86_64-unknown-uefi` (`mk/20-build.mk:89`), and adds `rust-src`
(`mk/20-build.mk:91`), clippy, and rustfmt (`mk/20-build.mk:94`); the stamp rule
itself is `mk/20-build.mk:85`.

`nonos-mk-check-deps` depends on the same toolchain stamp
(`mk/10-qemu.mk:148`). The default `nonos-mk` target builds the
microkernel-capsules baseline through `nonos-mk-capsules` (`mk/10-qemu.mk:135`).

## 3. Userland libc

The build compiles `userland/libc` into
`userland/libc/target/x86_64-nonos-user/release/libnonos_libc.a`
(`USERLAND_LIBC`, `mk/20-build.mk:168`). The recipe runs cargo with the pinned
toolchain, the `../x86_64-nonos-user.json` target, and `-Zbuild-std=core`
(`mk/20-build.mk:208`).

## 4. Capsule compilation

Each `userland/<capsule>/Capsule.mk` is included from `mk/20-build.mk`
(the include list begins at `mk/20-build.mk:407`). The shared capsule macro
snapshots identity and output paths, including the binary path, certificate path,
manifest path, key paths, handle, domain, namespace, endpoints, required caps,
optional caps, version, target, and source list (`nonos-mk/capsule.mk:94`).

The capsule ELF rule depends on `USERLAND_LIBC`, `Capsule.mk`, `Cargo.toml`,
`Cargo.lock`, capsule sources, shared userland library sources, and extra deps.
It runs cargo release build with the pinned toolchain, target
`../x86_64-nonos-user.json`, `-Zbuild-std=core,alloc`, and
`compiler-builtins-mem` when configured (`nonos-mk/capsule.mk:182`).

```
  Capsule.mk
      |
  nonos-mk/capsule.mk
      |
      +-- target/x86_64-nonos-user/release/<bin>
      +-- nonos-data/trust/capsules/<bin>.nonos_id_cert.bin
      +-- nonos-data/trust/capsules/<bin>.manifest.bin
```

## 5. Kernel builds

Kernel builds use `KERNEL_BUILD_FLAGS`: release profile, target
`x86_64-nonos.json`, `-Zbuild-std=core,alloc`, and
`compiler-builtins-mem` (`mk/20-build.mk:658`). Every profile target routes
through one `nonos_kernel_build` macro (`mk/20-build.mk:670`). `nonos-mk-check`
runs cargo check with `microkernel-core` only (`mk/20-build.mk:694`).

The desktop production build depends on signed artifact triples for core
services, drivers, network, desktop, apps, attest, and power, then runs
`nonos-mk-verify-desktop-gui-capsules` before building the kernel with feature
`microkernel-desktop-gui` (`mk/20-build.mk:984`). The larger image is
`microkernel-full-gui`, built by `nonos-mk-zerostate`
(`mk/20-build.mk:1014`); it is `microkernel-desktop-gui` plus the remaining
production hardware driver capsules (`Cargo.toml:586`). Two of those extras are
the input drivers that matter on real laptops: `microkernel-full-gui` pulls in
`nonos-capsule-driver-i2c-pci` and `nonos-capsule-driver-i2c-hid`
(`Cargo.toml:597`), which `microkernel-desktop-gui` does not carry. A kernel built
`desktop-gui` therefore has no i2c input path, which is the correct choice for
QEMU (PS/2 and xHCI cover it there) but wrong for hardware whose touchpad or
keyboard is behind i2c-HID.

`nonos-mk-verify-fast` runs static gates only. `nonos-mk-verify` runs static
gates, trust verification, and the symbol scan. `nonos-mk-test` adds the
required QEMU boot harnesses (`mk/40-run.mk:354`).

## 6. Build ordering

The kernel embeds every capsule's signed ELF, certificate, and manifest at
compile time with `include_bytes!`. The desktop shell mirror pulls its three
artifacts in from the build and trust locations
(`src/userspace/capsule_desktop_shell/embed.rs:18`), and a driver does the same
(`src/hardware/ps2_kbd_capsule/embed.rs:24`). Because `include_bytes!` resolves
at kernel compile time, those files must already exist and be current when the
kernel builds. The production recipes encode this ordering in their
prerequisites: `nonos-mk-desktop-gui-prod` lists every capsule's `*_ARTIFACTS`
(the signed ELF, cert, and manifest) as a dependency before the kernel build step
runs (`mk/20-build.mk:984`), so the capsules are compiled and signed first and the
kernel links against fresh bytes. A kernel built against a stale or missing
capsule artifact is the failure this ordering exists to prevent.

Release kernels also require a signing key. The `-prod` recipes depend on
`nonos-mk-ensure-signing-key` (`mk/10-qemu.mk:164`), which materialises the Ed25519
seed (`mk/10-qemu.mk:150`) and the ML-DSA-65 keypair (`mk/10-qemu.mk:156`), and the
kernel build itself passes `NONOS_SIGNING_KEY` into cargo (`nonos_kernel_build`
macro, `mk/20-build.mk:672`). The featureless kernel artifact rule lists the
signing key as an order-only prerequisite (`mk/20-build.mk:688`), so a build
without a key does not silently produce an unsigned kernel.

## 7. Troubleshooting

The build failures worth naming ahead of time all come from the ordering and key
requirements above.

**A kernel with no drivers.** If the desktop comes up but nothing responds to a
laptop's built-in keyboard or touchpad, the likely cause is the wrong profile:
`microkernel-desktop-gui` carries no i2c input drivers, only PS/2 and xHCI. Build
`microkernel-full-gui` (`nonos-mk-zerostate`, `mk/20-build.mk:1014`) for hardware
whose input is behind i2c-HID. More broadly, a driver capsule that is not in the
selected profile's feature set is simply not embedded, so its device is
unclaimed; the fix is to build the profile that includes it, not to change the
kernel.

**Missing publisher key.** Each capsule signs with its own publisher keypair, and
the per-capsule `nonos-mk-check-<slug>-keys` target asserts the Ed25519 and
ML-DSA-65 seeds and public files exist before signing
(`nonos-mk/capsule.mk:191`). A missing file fails with an explicit
`::error::missing <path>` and the exact `capsule-sign keygen` command to generate
it (`nonos-mk/capsule.mk:195`). This gates the capsule sign step, which gates the
kernel build that embeds it.

**Signing key not found.** A `*-prod` build with no kernel signing material stops
at `nonos-mk-ensure-signing-key` rather than producing an unsigned image. The
Ed25519 seed is generated from `/dev/urandom` on first build (`mk/10-qemu.mk:150`)
and the ML-DSA-65 keypair via `capsule-sign keygen` (`mk/10-qemu.mk:156`); if the
recipe that consumes `NONOS_SIGNING_KEY` runs and the key path is absent, the
kernel artifact rule's signing-key prerequisite (`mk/20-build.mk:688`) is what
forces it to exist first. On a fresh checkout this means the first production
build spends a step creating keys before it compiles the kernel.

**A `-Zbuild-std` or missing-`rust-src` failure.** Capsule and kernel builds both
pass `-Zbuild-std`, which needs the pinned toolchain with `rust-src` added. The
userland build-std recipes take an order-only dependency on the toolchain stamp
(`mk/20-build.mk:208`) precisely so this is installed first; a build-std error
usually means the toolchain stamp step, which adds `rust-src` (`mk/20-build.mk:91`),
has not run; `nonos-mk-check-deps` (`mk/10-qemu.mk:148`) depends on that stamp
precisely to guarantee it.

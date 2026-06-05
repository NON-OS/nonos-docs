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
disables the red zone, links `-no-pie`, sets relocation model `static`, and
uses code model `kernel` (`x86_64-nonos.json:16`).

| Target file | Intended binary | Link posture |
|-------------|-----------------|--------------|
| `userland/x86_64-nonos-user.json` | Capsule ELF | PIE, PIC, no default libraries |
| `x86_64-nonos.json` | Kernel ELF | static, kernel code model, no default libraries |

## 2. Toolchain bootstrap

The Makefile pins Rust to `nightly-2026-01-16` (`Makefile:64`). The toolchain
stamp target installs the pinned toolchain, adds `x86_64-unknown-uefi`, and
adds `rust-src`, clippy, and rustfmt (`Makefile:225`).

`nonos-mk-check-deps` depends on the same toolchain stamp
(`Makefile:176`). The default `nonos-mk` target builds the microkernel-capsules
baseline through `nonos-mk-capsules` (`Makefile:162`).

## 3. Userland libc

The Makefile builds `userland/libc` into
`userland/libc/target/x86_64-nonos-user/release/libnonos_libc.a`
(`Makefile:251`). The recipe runs cargo with the pinned toolchain, the
`../x86_64-nonos-user.json` target, and `-Zbuild-std=core`
(`Makefile:282`).

## 4. Capsule compilation

Each `userland/<capsule>/Capsule.mk` is included by the root Makefile
(`Makefile:345`). The shared capsule macro snapshots identity and output paths,
including the binary path, certificate path, manifest path, key paths, handle,
domain, namespace, endpoints, required caps, optional caps, version, target,
and source list (`nonos-mk/capsule.mk:91`).

The capsule ELF rule depends on `USERLAND_LIBC`, `Capsule.mk`, `Cargo.toml`,
`Cargo.lock`, capsule sources, shared userland library sources, and extra deps.
It runs cargo release build with the pinned toolchain, target
`../x86_64-nonos-user.json`, `-Zbuild-std=core,alloc`, and
`compiler-builtins-mem` when configured (`nonos-mk/capsule.mk:151`).

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
`compiler-builtins-mem` (`Makefile:517`). `nonos-mk-check` runs cargo check
with `microkernel-core` only (`Makefile:533`).

The desktop production build depends on signed artifact triples for core
services, drivers, network, desktop, apps, attest, and power, then runs
`nonos-mk-verify-desktop-gui-capsules` before building the kernel with feature
`microkernel-desktop-gui` (`Makefile:1086`).

`nonos-mk-verify-fast` runs static gates only. `nonos-mk-verify` runs static
gates, trust verification, and the symbol scan. `nonos-mk-test` adds the
required QEMU boot harnesses (`Makefile:1366`).

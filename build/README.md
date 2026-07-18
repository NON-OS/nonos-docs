# Build

This section describes how NØNOS builds capsule binaries, signs them, and links
them into production kernel images. Read
[Architecture Overview](../architecture/overview.md) first.

| Page | Scope |
|------|-------|
| [Quickstart](quickstart.md) | From a clean clone to your own signed, attested capsule running on the desktop. |
| [Configuring a kernel](menuconfig.md) | Profiles, `make menuconfig`, `.nonos-config`, and the security options a build chooses. |
| [Toolchain](toolchain.md) | Rust target files, cargo flags, Make targets, and capsule compilation. |
| [Signing](signing.md) | Capsule certificates, manifests, trust policy, and embedded artifacts. |
| [Build and Verification Workflows](workflows.md) | Static, trust, symbol, QEMU, boot-harness, and change-type verification workflows. |

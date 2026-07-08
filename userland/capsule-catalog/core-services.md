# Core Service Capsules

The core services are the capsules the rest of the system depends on: the filesystem, the key store, the
entropy and crypto pools, policy, and the install, payment, and power rails. Each now has a dedicated,
verified deep page covering its server loop, wire protocol, per-operation logic, state model, security,
and honest gaps. This page indexes them.

| Capsule | Service | Caps | What it is |
|---------|---------|------|-----------|
| [vfs](vfs.md) | `vfs_pool` :4104 | `0x19` | The application filesystem: 15-op store, per-caller FD table, caller attestation, `/capsules` read-only. |
| [ramfs](ramfs.md) | `ramfs` :4096 | `0x38` | The `/ram` filesystem, per-file ChaCha20-Poly1305 encryption with fresh-nonce writes. |
| [keyring](keyring.md) | `keyring` :4098 | `0x39` | The key store and wallet signer: owner-pid isolation, secure-wipe on drop, ETH/NOX signing. |
| [entropy](entropy.md) | `entropy_pool` :4100 | `0x39` | The RDRAND-backed random-bytes service with observability counters. |
| [crypto](crypto.md) | `crypto_pool` | `0x39` | The stateless crypto compute pool with per-op caps and request-buffer wipe. |
| [policy](policy.md) | `policy` :4108 | `0x219` | The typed config store; reads open, writes gated to the settings apps. |
| [attest](attest.md) | `attest` :4444 | `0x19` | System info and stated invariants. Honestly: text claims and a boot label, not cryptographic proofs. |
| [installer](installer.md) | `installer` :4112 | `0x19` | Loads capsules through the kernel's verified-load syscall; trust is the kernel's. |
| [payment](payment.md) | `payment` :4110 | `0x19` | Issues keyring-signed receipts with per-payer nonces and a drain queue. |
| [market](market.md) | `market.index` :4106 | `0x19` | Serves the signed app index; real Ed25519 verifier (reject-all stub under `offline-verify`). |
| [power](power.md) | `power` :4448 | `0x219` | Reboot and shutdown through the kernel admin syscalls. |
| [process-manager](process-manager.md) | `app.process_manager` :4730 | `0x1819` | A GUI viewer of running services and CPU usage via `mk_proc_stat`. |

Two honest notes carried from the deep pages: the [attest](attest.md) capsule returns human-authored
invariant claims and a boot label, not cryptographic proofs (the real
[proof system](../../subsystems/proof-system/README.md) is in the kernel), and the [installer](installer.md)
and [market](market.md) have an `offline-verify` build feature that swaps the real verifier for a
development reject-all stub.

The shared model, service registration, verified spawn, and the IPC request loop, is the
[userland overview](../README.md); the flat inventory of every handle and port is the
[capsule inventory](../capsules.md).

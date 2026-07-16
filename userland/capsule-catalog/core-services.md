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

## Security analysis

Read the mask column as a least-privilege statement and a pattern falls out. The baseline for a service is
`0x19`, CoreExec (1), IPC (8), Memory (16), and five capsules run on exactly that and nothing more: vfs,
attest, installer, payment, and market. None of them holds a hardware capability (Driver, Mmio, Irq, Dma,
Pio), so no filesystem, install, payment, or market bug can reach a device. The Crypto bit (32) is added
only where a capsule genuinely computes crypto, giving `0x39` for keyring, entropy, and crypto, and the
[ramfs](ramfs.md) mask `0x38` is that set minus CoreExec. The Admin bit (512) is the one elevated power in
the fleet, and it appears in exactly two masks, `0x219` for [policy](policy.md) and [power](power.md), the
capsules that push a kernel security toggle and reset the machine respectively; in both, Admin is the only
capability beyond the service baseline. The one GUI member, [process-manager](process-manager.md), carries
`0x1819`, the baseline plus the two graphics bits it needs to paint, and no authority over the processes it
observes. So no core service holds FileSystem it does not use, no service holds a hardware capability, and
the only Admin-class power in the set is confined to the two capsules whose whole job is a privileged verb.
The load-bearing caveat, carried from the deep pages, is that [policy](policy.md)'s write gate is by service
name rather than a fine-grained capability, a real trust concentration on the settings apps.

## Debugging

Every core service that the kernel init fleet spawns prints a boot marker through `capsule_boot::boot`
(`src/userspace/init/capsule_boot/run.rs`), which emits `boot_log::ok(prefix, "capsule spawned")` on
success and a `[ERROR]` line with the `SpawnError` on failure. On a machine with no serial port a
`NONOS_FBCONSOLE=1` build mirrors these to the framebuffer. The first question for any core service, did it
load, is that marker; the second, did it register, is whether a caller's `mk_service_lookup` on its name
resolves.

```
  [RAMFS] capsule spawned                spawn_plan/core.rs:19               ramfs :4096
  [VFS] capsule spawned                  spawn_plan/core.rs:32               vfs_pool :4104
  [MARKET] capsule spawned               spawn_plan/core.rs:39               market.index :4106
  [KEYRING] capsule spawned              spawn_plan/core.rs:48               keyring :4098
  [ENTROPY] capsule spawned              spawn_plan/core.rs:54               entropy_pool :4100
  [CRYPTO] capsule spawned               spawn_plan/core.rs:60               crypto_pool :4102
  [POLICY] capsule spawned               spawn_plan/core.rs:67               policy :4108
  [ATTEST] capsule spawned               spawn_plan/desktop_services.rs:29   attest :4444
  [INSTALLER] capsule spawned            spawn_plan/desktop_services.rs:37   installer :4112
  [APP-PROCESS-MANAGER] capsule spawned  spawn_plan/apps_tools.rs:50         app.process_manager :4730
```

Two of the rails in the table above are not in this list on purpose: [payment](payment.md) (:4110) and
[power](power.md) (:4448) are built into the image (their `Capsule.mk` is included by the Makefile) but are
not spawned by the kernel init spawn plan, so they emit no boot marker in the fleet. For those two the test
that they are up is a resolving `mk_service_lookup`, not a boot line, and for payment the practical tell is
that the installer's paid path returns `EAGAIN` when it cannot reach the payment service. Each deep page
carries that capsule's own request-time failure signatures.

The shared model, service registration, verified spawn, and the IPC request loop, is the
[userland overview](../README.md); the flat inventory of every handle and port is the
[capsule inventory](../capsules.md).

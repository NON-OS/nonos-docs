# Capsule SDK

This page describes the userland SDK surface: `nonos_libc`, `nonos_abi`,
runtime crates, and the app skeleton used by GUI applications. Read
[Userland Model](README.md) first, then [Syscall ABI Reference](../abi/syscalls.md).

Read this page bottom-up when debugging a crash and top-down when writing a
capsule. Bottom-up follows raw syscall entry to the kernel dispatcher. Top-down
starts with `_start`, runtime setup, the service loop or app skeleton, and then
the binding calls.

---

## 1. Crate layers

The low-level syscall surface is split into small no_std crates.
`nonos_abi` is no_std, exports `syscall`, `syscall_diverging`, input helpers,
memory mapping helpers, and syscall number constants (`userland/nonos_abi/src/lib.rs:17`).
On x86_64, its raw syscall wrapper moves the syscall number into `rax`, shifts
arguments into the kernel ABI register order, executes `syscall`, and returns
through `rax` (`userland/nonos_abi/src/raw.rs:17`).

`nonos_libc` is also no_std and is the wider capsule-facing binding layer
(`userland/libc/src/lib.rs:17`). It re-exports process, memory, IPC, broker,
crypto, graphics, surface, input, battery, admin, time, and debug bindings from
one crate (`userland/libc/src/lib.rs:36`).

`nonos_runtime` provides the generic run wrapper. It calls runtime boot, exits
with code 1 on boot failure, calls the capsule entry function, runs cleanup,
and exits with code 0 (`userland/nonos_runtime/src/run.rs:21`).

```
  +-----------------------------+
  | capsule code                |
  | _start or app_skeleton::run |
  +--------------+--------------+
                 |
  +-----------------------------+
  | nonos_runtime               |
  | boot, entry, cleanup, exit  |
  +--------------+--------------+
                 |
  +-----------------------------+
  | nonos_libc                  |
  | typed syscall bindings      |
  +--------------+--------------+
                 |
  +-----------------------------+
  | nonos_abi                   |
  | raw syscall instruction     |
  +--------------+--------------+
                 |
  +-----------------------------+
  | kernel syscall dispatcher   |
  +-----------------------------+
```

## 2. Service capsule shape

A service capsule is a no_std, no_main binary with an `_start` entry point. The
compositor example initializes the heap, waits for setup, registers the service
name `compositor` on port `4310`, then enters its server loop
(`userland/compositor/src/main.rs:31`). The server loop allocates request and
reply buffers, drains IPC, ticks the frame pacer, waits for vsync, and yields if
vsync wait fails (`userland/compositor/src/server/runner/entry.rs:23`).

Service setup is capsule-specific. The WM setup resolves the compositor port,
probes compositor health, queries display info, and builds a context with
window table, focus model, z stack, subscriptions, and request id state
(`userland/capsule_wm/src/setup/run.rs:36`).

```
+--------------------------+
| service _start           |
+------------+-------------+
             |
+------------+-------------+
| heap setup               |
| capsule setup            |
+------------+-------------+
             |
+------------+-------------+
| service registration     |
| server run               |
+------------+-------------+
             |
+------------+-------------+
| recv frame               |
| dispatch handler         |
+------------+-------------+
             |
+------------+-------------+
| reply or wait            |
+--------------------------+
```

## 3. GUI app skeleton

`nonos_app_skeleton` is no_std and exports the app trait, manifest, input
types, paint buffer, clipboard helpers, and `run` entry point
(`userland/app_skeleton/src/lib.rs:17`). An app implements three methods:
`manifest`, `on_event`, and `paint` (`userland/app_skeleton/src/app/behavior.rs:21`).

The app manifest carries the window title, window id, window kind, initial
position, width, height, and input kind mask
(`userland/app_skeleton/src/app/manifest.rs:19`).

```
+--------------------------+
| App implementation       |
+------------+-------------+
             |
+------------+-------------+
| manifest                 |
| title id geometry input  |
+------------+-------------+
             |
+------------+-------------+
| on_event                 |
| event outcome            |
+------------+-------------+
             |
+------------+-------------+
| paint                    |
| pixels in PaintBuffer    |
+--------------------------+
```

The runner initializes the heap, resolves required peers, builds the app,
opens and primes the window, then services frames forever
(`userland/app_skeleton/src/runner/entry.rs:30`). Booting an app opens a window,
subscribes for input, and primes the first frame
(`userland/app_skeleton/src/runner/boot.rs:33`).

## 4. Window setup

Opening a window allocates backing memory, registers and shares the surface,
announces the window to WM, and returns a binding with surface handle, backing
address, placement, stride, and byte length (`userland/app_skeleton/src/setup/open.rs:26`).
Surface registration uses `mk_surface_register` with ARGB8888 metadata, then
uses `mk_surface_share` to obtain the share handle
(`userland/app_skeleton/src/setup/register.rs:21`).

After WM placement, the app skeleton submits the scene to the compositor with
app layer z value `2` (`userland/app_skeleton/src/setup/submit_scene.rs:24`).
Input subscription sends the app's kind mask to input_router and retries up to
four times (`userland/app_skeleton/src/setup/subscribe_input.rs:22`).

## 5. Developer contract

| Developer writes | SDK provides | Source |
|------------------|--------------|--------|
| `_start` service loop | `nonos_libc` syscall bindings | `userland/libc/src/lib.rs:36` |
| `App` implementation | App runner, peer discovery, window open, input subscribe | `userland/app_skeleton/src/runner/entry.rs:30` |
| Manifest metadata | Window title, id, kind, position, size, input mask | `userland/app_skeleton/src/app/manifest.rs:19` |
| Paint code | Mutable `PaintBuffer` passed to `paint` | `userland/app_skeleton/src/app/behavior.rs:21` |

## 6. The Capsule.mk contract

Source code alone does not make a capsule. A capsule is the signed, attested
package the kernel will verify: an ELF, a NØNOS-ID certificate, a manifest, and a
STARK attestation trailer. All four are produced by the shared build/sign macro,
which a per-capsule `userland/<capsule>/Capsule.mk` drives by declaring the
capsule's identity in `CAPSULE_*` variables and then including the macro
(`nonos-mk/capsule.mk:1`). The macro errors out if any required variable is
missing, so the declaration is the single source of truth for the capsule's
identity and authority (`nonos-mk/capsule.mk:29`).

`userland/capsule_hello/Capsule.mk` is the minimal example:

```
CAPSULE_SLUG             := hello
CAPSULE_HANDLE           := app.hello
CAPSULE_DOMAIN           := systems.nonos
CAPSULE_NAMESPACE        := systems.nonos.app.hello
CAPSULE_SERVICE_ENDPOINT := service:4810:app.hello
CAPSULE_REPLY_ENDPOINT   := reply:4811:endpoint.app.hello.reply
# CoreExec|IPC|Memory|GraphicsDisplayQuery|GraphicsSurfaceCreate
CAPSULE_REQUIRED_CAPS    := 0x1819
include nonos-mk/capsule.mk
```

| Variable | Meaning | Source |
|----------|---------|--------|
| `CAPSULE_SLUG` | Short name; namespaces the generated make targets and variables | `nonos-mk/capsule.mk:29` |
| `CAPSULE_HANDLE` / `CAPSULE_DOMAIN` / `CAPSULE_RECOVERY` | Hashed to the `nonos_id` by `capsule-sign derive-id` | `nonos-mk/capsule.mk:201` |
| `CAPSULE_NAMESPACE` | Reverse-domain namespace signed into the cert and manifest | `nonos-mk/capsule.mk:44` |
| `CAPSULE_SERVICE_ENDPOINT` / `CAPSULE_REPLY_ENDPOINT` | `kind:port:name` triples declared in the signed manifest | `nonos-mk/capsule.mk:47` |
| `CAPSULE_INSTANCE_ENDPOINTS` | Extra `kind:port:name` triples for runtime-spawned windows | `nonos-mk/capsule.mk:126` |
| `CAPSULE_REQUIRED_CAPS` | Hex `u64` mask of caps the capsule needs | `nonos-mk/capsule.mk:56` |
| `CAPSULE_OPTIONAL_CAPS` | Hex mask of caps taken only if the spawn grant offers them (default `0x0`) | `nonos-mk/capsule.mk:76` |
| `CAPSULE_CAPS_CEILING` | Cert ceiling; defaults to `CAPSULE_REQUIRED_CAPS` | `nonos-mk/capsule.mk:77` |

The `CAPSULE_REQUIRED_CAPS` mask is checked against the capsule's `.nonos.caps`
ELF section before every manifest signature, so a mask that disagrees with the
compiled caps fails the sign step rather than shipping (`mk/00-config.mk:59`).

## 7. Build, sign, and verify

Including the macro materializes a standard target set, each namespaced by slug
(`nonos-mk/capsule.mk:6`). Before signing, the publisher key pair must exist:
seeds live in `.keys/` (gitignored) and public keys in
`nonos-data/trust/keys/`. Generate a pair with the host signing tool:

```
nonos-sign/target/release/capsule-sign keygen --alg ed25519 --out .keys/<bin>_publisher
nonos-sign/target/release/capsule-sign keygen --alg mldsa65 --out .keys/<bin>_publisher
```

`nonos-mk-check-<slug>-keys` asserts all four key files are present and prints
that exact keygen command if any is missing (`nonos-mk/capsule.mk:190`).

| Target | Effect | Source |
|--------|--------|--------|
| `make nonos-mk-<slug>` | Build the userland ELF with `-Zbuild-std=core,alloc` against `x86_64-nonos-user.json` | `nonos-mk/capsule.mk:163` |
| `make nonos-mk-<slug>-sign` | Produce the cert, manifest, and attestation trailer | `nonos-mk/capsule.mk:282` |
| `make nonos-mk-<slug>-verify` | Re-verify the artifacts against the trust-anchor policy | `nonos-mk/capsule.mk:285` |
| `make nonos-mk-check-<slug>-keys` | Assert the publisher seeds and pubs are present | `nonos-mk/capsule.mk:190` |

The build is pinned to toolchain `nightly-2026-01-16` (`mk/00-config.mk`),
compiles `no_std` against the `x86_64-nonos-user.json` target, and passes
`-Zbuild-std` per capsule so `core` and `alloc` are rebuilt for the user target
(`nonos-mk/capsule.mk:170`).

Signing is three tool invocations, all driven by the macro so the values stay in
sync:

1. `capsule-sign sign-id-cert` writes the hybrid NØNOS-ID certificate: it binds
   the derived `nonos_id`, the namespace glob, the caps ceiling, and the
   Ed25519 + ML-DSA-65 publisher public keys, signed by the trust-anchor seeds
   (`nonos-mk/capsule.mk:214`).
2. `capsule-sign sign-manifest` writes the schema-v3 manifest: it hashes the ELF
   into `payload_hash`, records the target triple, the required and optional cap
   masks, and every declared endpoint, then dual-signs with the publisher seeds
   (`nonos-mk/capsule.mk:230`). The manifest depends on the ELF, so rebuilding
   the binary forces a re-sign and `payload_hash` can never drift from the bytes.
3. The macro immediately calls `capsule-sign verify-manifest` against the cert
   and the trust-anchor policy, failing the build if the freshly signed manifest
   does not verify (`nonos-mk/capsule.mk:244`).

The attestation trailer is not a per-capsule proof. The whole capsule set is
enrolled at once by the transparent STARK enrollment, which writes the policy
root and every trailer together so the root commits to the real capsule
measurements; each trailer therefore depends only on that enrollment step
(`nonos-mk/capsule.mk:252`). The retired curve-based per-capsule attestation is
kept disabled in the same file for reference; it was not post-quantum and its
trailer was incompatible with the STARK spawn gate (`nonos-mk/capsule.mk:258`).

The signed artifacts land in the committed trust bundle: the cert and manifest
under `nonos-data/trust/capsules/<bin>.nonos_id_cert.bin` and `.manifest.bin`,
the trailer as `.zk_trailer.bin`, so a clean checkout already carries everything
the kernel verifier needs (`nonos-mk/capsule.mk:110`).

## 8. Install and run

A signed capsule reaches the runqueue one of two ways. At build time the kernel
mirror under `src/userspace/capsule_<name>` embeds the ELF, cert, and manifest
with `include_bytes!` and exposes a spawn function that hands a
`CapsuleSpecVerified` to `spawn_verified`; init spawns the fleet in order at boot
(see [Userland Model](README.md), sections 1 and 4). At runtime the installer
capsule marshals the same four blobs and hands them to the `mk_capsule_load`
syscall, which runs the full trust chain before anything spawns
(see [Installer](installer/README.md)). Either path ends at the same verified
spawn gate: nothing runs that was not decoded, signature-checked, hash-matched,
and cap-bounded first.

| Status tag | Meaning for this SDK |
|------------|----------------------|
| IMPLEMENTED / ENFORCED | The `Capsule.mk` contract, the four generated targets, and the hybrid sign + verify flow are in-tree and run on every build. |
| PROVEN | The manifest verifier's cap-grant rule (`required \| (optional & granted)`, ceiling check) and the signature path are the kernel gate; the STARK enrollment produces the attestation the spawn gate checks. |
| PARTIAL | Runtime install through `mk_capsule_load` is wired for the installer; general third-party install-on-demand is not a general-user flow yet. |

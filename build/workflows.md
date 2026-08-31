# Build and Verification Workflows

This page describes the operational build and verification workflows. Read
[Toolchain](toolchain.md) and [Signing](signing.md) first.

The distinction matters: static checks are not runtime proof. A production
change needs the right combination of static gates, signed capsule verification,
symbol scan, and QEMU boot harnesses.

---

## 1. Workflow map

The build is split by concern into `mk/*.mk` (config, qemu, build, image, run, ci), included by the
top-level `Makefile`; the target line numbers below are in those files, not in `Makefile`.

| Workflow | Target | What it proves |
|----------|--------|----------------|
| Baseline build | `nonos-mk` | Builds the microkernel capsule baseline through `nonos-mk-capsules` and prints the next packaging, run, verify, and test targets (`mk/10-qemu.mk:135`). |
| Toolchain bootstrap | `nonos-mk-toolchain` | Creates the toolchain stamp (`mk/10-qemu.mk:146`); the stamp rule installs the pinned toolchain, adds the `x86_64-unknown-uefi` target, and installs `rust-src`, clippy, and rustfmt (`mk/20-build.mk:85`, add-uefi `:89`, rust-src `:91`). |
| Userland libc | `nonos-mk-libc` | Builds `userland/libc` for `x86_64-nonos-user` with `-Zbuild-std=core` (`mk/20-build.mk:216`, recipe `mk/20-build.mk:208`). |
| Capsule build and sign | `nonos-mk-<slug>`, `nonos-mk-<slug>-sign` | Builds the capsule ELF (`nonos-mk/capsule.mk:182`), asserts publisher keys (`:191`), signs the NØNOS-ID certificate (`:211`), signs the manifest (`:231`), and re-verifies the manifest against the certificate and trust policy (`:245`). |
| Configure a kernel | `make menuconfig`, `make from-config` | `menuconfig` walks the profiles and the security options with `tools/nonos-config` and writes `.nonos-config` (`mk/20-build.mk:806`); `from-config` builds exactly that profile and feature set (`mk/20-build.mk:813`). This is where a user assembles their own kernel. |
| Full system image | `nonos-mk-zerostate` | The canonical production image: every capsule and driver, the transparent STARK spawn gate enforced (`microkernel-full-gui,nonos-stark-attest`), dual Ed25519 + ML-DSA-65 signing, the anti-rollback index bound into the signature, and the TPM anti-rollback floor (`mk/20-build.mk:1014`). |
| Desktop production image | `nonos-mk-desktop-gui-prod` | Requires signed artifacts for core services, drivers, network, desktop, first-party apps, attest, and power, then builds the `microkernel-desktop-gui` profile with the STARK gate (`mk/20-build.mk:984`). Every profile target routes through one `nonos_kernel_build` macro (`mk/20-build.mk:670`), so the build command exists in a single place. |
| ESP packaging | `nonos-mk-esp` | Builds bootloader and attested kernel, copies them into the ESP, writes `boot.cfg`, writes `startup.nsh`, and reports the ESP directory (`mk/20-build.mk:1131`, kernel copy `:1148`, `boot.cfg` `:1149`, `startup.nsh` `:1150`). |
| QEMU desktop run | `nonos-mk-run` | Builds the ZeroState production image, packages ESP, creates block image and OVMF vars, starts a software TPM, boots QEMU with block, GPU, USB, RNG, TPM, serial, and no reboot (`mk/40-run.mk:75`, qemu line `:81`). Network is NAT by default; `nonos-mk-run-net` enables explicit QEMU host forwarding. |
| Static lane | `nonos-mk-verify-fast` | Runs static checks only through `nonos-mk-static` (`mk/40-run.mk:346`, `:303`). |
| Full verify lane | `nonos-mk-verify` | Runs static checks, production desktop trust verification, and microkernel symbol scan (`mk/40-run.mk:349`; trust `mk/20-build.mk:375`; scan `mk/40-run.mk:318`). |
| Full test lane | `nonos-mk-test` | Runs full verify plus RAMFS, keyring, and desktop GUI boot harnesses (`mk/40-run.mk:354`). |

```
+----------------+
| source change  |
+-------+--------+
        |
+-------+--------+
| static lane    |
+-------+--------+
        |
+-------+--------+
| signed build   |
| trust verify   |
+-------+--------+
        |
+-------+--------+
| symbol scan    |
+-------+--------+
        |
+-------+--------+
| QEMU harnesses |
+----------------+
```

## 2. Capsule artifact workflow

Each `userland/<capsule>/Capsule.mk` declares identity and includes the shared
capsule macro `nonos-mk/capsule.mk`, which requires the identity vars to be set
(`nonos-mk/capsule.mk:29`) and materializes the standard target set documented in
its header (`nonos-mk/capsule.mk:4`): build the ELF, sign cert and manifest,
check publisher keys, and expose binary, cert, manifest, and artifact paths.

Metadata is snapshotted at include time so later capsule includes cannot clobber
earlier capsule values. The snapshot covers directory, binary name, target,
binary path, cert path, manifest path, stale marker, publisher key paths, handle,
domain, namespace, service endpoint, reply endpoint, required caps, optional
caps, and cap ceiling (`nonos-mk/capsule.mk:94`-`127`).

The ELF rule depends on userland libc, the capsule Makefile, Cargo metadata,
capsule sources, shared userland sources, and extra deps, then builds with the
pinned toolchain and the `x86_64-nonos-user` target
(`nonos-mk/capsule.mk:182`, `-Zbuild-std` `:184`).

The cert rule recomputes the NØNOS identity from handle, domain, and recovery
(`derive-id`, `nonos-mk/capsule.mk:202`), then signs with serial, namespace glob,
caps ceiling, trust-anchor epoch, validity window, publisher public keys,
trust-anchor seeds, metadata, and output path (`sign-id-cert`,
`nonos-mk/capsule.mk:211`).

The manifest rule depends on the ELF, cert, capsule Makefile, and signing tool.
It signs namespace, version, target, ELF path, required caps, optional caps,
service endpoint, reply endpoint, publisher seeds, and output path
(`sign-manifest`, `nonos-mk/capsule.mk:231`), then verifies the manifest against
cert and policy (`verify-manifest`, `nonos-mk/capsule.mk:245`).

## 3. Static workflow

`nonos-mk-static` first runs the capability-parity and attestation-parameter
guards (`nonos-mk-check-caps`, `mk/40-run.mk:299`) and then
`nonos-ci/run-static-checks.sh` (`mk/40-run.mk:303`). The script is the local and
CI static gate.

The static script checks the active Cargo default profile, rejects an active
legacy `nonos = [...]` profile, and runs the feature-profile checker. It also
gates architecture leaks, deprecated memory shims, deleted Linux-shaped syscall
paths, kernel-resident service engines, fake userspace service directories,
unexpected kernel driver trees, capsule README contract coverage, kernel service
directory shape, surface/input syscall dispatch, wire shape agreement, submodule
cleanliness, and a final pass/fail exit. The checks live in
`nonos-ci/run-static-checks.sh`.

This workflow does not boot the OS. It is necessary, but not sufficient, for a
runtime change.

## 4. Trust and symbol workflow

`nonos-mk-verify` first runs static checks, then calls `nonos-mk-verify-trust`,
then calls `nonos-mk-scan` (`mk/40-run.mk:349`). Trust verification builds the
desktop GUI production profile, runs host trust chain tests, runs on-disk
artifact tests, and verifies the baked trust artifact SHA-256 ledger
(`nonos-mk-verify-trust`, `mk/20-build.mk:375`).

The symbol scan requires the microkernel binary, dumps symbols with `nm`, checks
for forbidden legacy symbol fragments, runs the CI scan script, and reports pass
when no legacy-tree symbols are found (`nonos-mk-scan`, `mk/40-run.mk:318`).
Forbidden fragments include old filesystem, storage, desktop, graphics, shell,
app service, agent service, and network service module path fragments
(`MICROKERNEL_FORBIDDEN_SYMBOLS`, `mk/40-run.mk:313`).

## 5. QEMU run workflow

The normal interactive run target depends on a software TPM start, the live
production ZK proof, the QEMU block image and store, writable OVMF vars, the
ZeroState kernel build, and ESP packaging (`mk/40-run.mk:75`). The target boots
QEMU with 2 GiB default memory, HVF, Q35, FAT-backed ESP, OVMF code, writable
OVMF vars, virtio block, virtio GPU, NAT networking, an xHCI controller (with
PS/2 i8042 for input under hvf), virtio RNG, the swtpm CRB device, audio, a QMP
control socket, serial, no VGA display, and no reboot (`mk/40-run.mk:81`, with
the device variables defined in `mk/10-qemu.mk`).

The serial-log run target uses the desktop GUI production build and ESP but runs
with serial output to a file and no display (`nonos-mk-run-serial-log`,
`mk/40-run.mk:168`; the interactive `nonos-mk-run-serial` is `mk/40-run.mk:153`).
The debug target builds the desktop GUI production image and starts QEMU with a
GDB stub on port `1234` (`nonos-mk-debug`, `mk/40-run.mk:190`).

## 6. Runtime evidence workflow

Runtime evidence must come from the production boot path, not from retired
harness-only profiles. A release claim needs a bounded serial capture, capsule
attestation receipt, hardware inventory where relevant, and an explicit statement
of which writable surfaces were present.

## 7. Required workflow by change type

| Change type | Minimum local workflow |
|-------------|------------------------|
| Documentation only | Citation/style checks for the edited docs. |
| Capsule source only | `make nonos-mk-<slug>` and `make nonos-mk-<slug>-sign`, then the relevant profile build. |
| Capsule identity, caps, endpoint, or signing metadata | Capsule signing plus trust-manifest verification. |
| Desktop GUI, input, WM, compositor, app skeleton | Production desktop GUI boot evidence and input-device transcript. |
| Storage capsule path | Source audit of mount, encryption, block I/O, flush, cleanup, and recovery paths. |
| Security, entropy, crypto, keyring | Trust verification plus source audit of key lifetime, entropy source, and capability gates. |
| Release candidate | Production boot evidence, hardware dossier, attestation receipts, and closed blockers from this page. |

## 8. Troubleshooting

The lanes above fail in a few characteristic ways, and the message tells you which
lane caught the problem.

**The static lane fails on an arch leak.** `nonos-mk-static` runs the CI static
script (`mk/40-run.mk:303`), which counts `cfg(target_arch)` and `crate::arch::x86_64::`
uses outside `src/arch/` and fails if they grow. A build that fails here has
generic code naming an architecture directly, the thing the
[architecture boundary](../arch/boundary.md) exists to prevent. The fix is to
route through `Arch::` or the appropriate seam, not to raise the baseline.

**The symbol scan fails on a forbidden symbol.** `nonos-mk-scan` dumps the
microkernel image with `nm` and fails if it finds a legacy module-path fragment
(old filesystem, storage, desktop, graphics, shell, or service-engine symbols),
`mk/40-run.mk:318`. A failure means a kernel-resident implementation of something
that should be a capsule leaked back into the image. The scan also fails outright
if the microkernel binary does not exist, printing the build-first hint
(`mk/40-run.mk:320`); that is an ordering problem, not a leak, so build the
baseline first.

**Trust verification fails.** `nonos-mk-verify-trust` builds the desktop GUI
production profile and then runs the host trust-chain tests, the on-disk artifact
tests, and the baked-artifact SHA-256 ledger check (`mk/20-build.mk:375`). A failure
here is a mismatch between what was signed and what the kernel would verify: a
stale or unsigned capsule artifact, a manifest that no longer verifies against its
certificate, or a baked trust file whose hash does not match the ledger. This is
the same class of failure the [signing](signing.md) page's troubleshooting covers,
surfaced by the verify lane rather than at capsule sign time.

**A lane passed but the OS still misbehaves at runtime.** This is the point
section 1 opens with: static checks are not runtime proof. `nonos-mk-verify` green
means the static gates, trust, and symbol scan all passed, but it does not boot
the OS. Only the QEMU harnesses in `nonos-mk-test` (`mk/40-run.mk:354`) and the
interactive run lanes exercise the running system. A change that compiles, signs,
and scans clean can still fault at boot; the runtime-evidence workflow in section
6 exists for exactly that gap.

**The run lane never reaches the desktop.** `nonos-mk-run` depends on the
software TPM, the ZeroState build, ESP packaging, a block image, and writable
OVMF vars (`mk/40-run.mk:75`). A boot that stalls before the desktop is usually a runtime
fault in a capsule or a driver that did not claim its device, not a build failure;
the serial log (`QEMU_SERIAL_LOG`, default `target/qemu-serial.log`) is the first
place to look, and the debug run lane with GDB on port 1234 is the second. A
desktop that comes up but ignores a laptop's built-in input is the profile
problem from the [toolchain](toolchain.md) page: `microkernel-desktop-gui` has no
i2c input drivers, `microkernel-full-gui` does.

## 9. Source map

```
  mk/20-build.mk
    :375   nonos-mk-verify-trust    build desktop-gui-prod, then host + on-disk + ledger trust checks
    :984   nonos-mk-desktop-gui-prod  signed-artifact deps, then the microkernel-desktop-gui kernel
    :1014  nonos-mk-zerostate       the full ZeroState image (microkernel-full-gui + nonos-stark-attest)
    :1131  nonos-mk-esp             bootloader + attested kernel into the ESP
  mk/40-run.mk
    :75    nonos-mk-run             swtpm, ZeroState build, ESP, block image, OVMF vars, then QEMU
    :303   nonos-mk-static          check-caps guards, then nonos-ci/run-static-checks.sh
    :318   nonos-mk-scan            nm symbol dump and forbidden-fragment scan
    :346   nonos-mk-verify-fast     static gates only
    :349   nonos-mk-verify          static + trust + scan
    :354   nonos-mk-test            verify + the required QEMU boot harnesses
  mk/10-qemu.mk                     QEMU/OVMF/swtpm discovery and device variables
  nonos-ci/run-static-checks.sh     the arch-leak, shape, and contract static gates
  nonos-mk/capsule.mk               the per-capsule sign and verify rules (see signing.md)
```

The capsule sign and verify rules these lanes depend on are on the
[signing](signing.md) page, the target files and build ordering are on the
[toolchain](toolchain.md) page, and the arch-leak gate enforces the boundary
documented under [arch/](../arch/boundary.md). The build is split into `mk/*.mk`
(config, qemu, build, image, run, ci), included by the top-level `Makefile`; the
anchors in this map are verified against those files.

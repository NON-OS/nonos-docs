# Packaging and installing a capsule: the `nonos` toolchain

The SDK gets you a binary. This layer gets that binary onto a running system as a
signed, attested capsule the kernel will agree to spawn. It is a small command
line tool, `nonos`, backed by a handful of libraries that each own one step of
the pipeline. The libraries live under `userland/platform/`; the tool that
drives them is `userland/platform/nonos_capsule/`.

The through-line to keep in mind: the capability bits your code declared in its
`.nonos.caps` section (see [../sdk/writing-apps.md](../sdk/writing-apps.md)) have
to match the manifest, the manifest gets signed, the signature gets verified at
install, and the kernel attests the whole thing again at spawn. The same
authority claim is checked at four points, and any mismatch stops the capsule
before it runs.

## The command line

`nonos_capsule/src/dispatch.rs` is the whole command surface, and it is worth
reading in full because it is short:

```
new       scaffold a new capsule project
build     compile it for the NONOS user target
manifest  produce or inspect the manifest
sign      sign the manifest and certificate
install   verify and install into the registry
run       build, sign, install, and launch
inspect   show what a built capsule declares
remove    uninstall
```

Each maps to one file under `nonos_capsule/src/cmd/`. There is no hidden state
and no daemon; the tool is a thin dispatcher over the libraries below, which is
what makes the pipeline auditable step by step.

## Build

`cmd/build.rs` loads the manifest, reads its `target`, and shells out to cargo
with the NONOS target JSON and the `no_std` build-std flags:

```
cargo build --release --target <target>.json \
    -Zbuild-std=core,alloc -Zbuild-std-features=compiler-builtins-mem
```

The target is taken from the manifest, overridable with `NONOS_TARGET_JSON`, and
the compiler is overridable with `NONOS_RUSTC`. That is the entire build step:
there is no custom build system, just cargo pointed at a bare-metal target, so a
capsule builds exactly the way any `no_std` Rust binary does.

## The manifest

`nonos_manifest` (`userland/platform/nonos_manifest/`) owns the format. A
`Manifest` (`userland/platform/nonos_manifest/src/model.rs`) is:

```rust
pub struct Manifest {
    pub name: String,
    pub namespace: String,
    pub cert: String,
    pub version: (u32, u32, u32),
    pub target: String,
    pub required_caps: Vec<String>,
    pub optional_caps: Vec<String>,
    pub endpoints: Vec<Endpoint>,
    pub pub_seeds: Vec<(String, String)>,
}
```

The two capability lists are the important part and they are not the same thing.
`required_caps` are what the capsule must be granted or it should not run;
`optional_caps` are what it will use if granted and do without otherwise. At
spawn the kernel installs `required | (optional & granted)`, so optional caps can
be withheld by policy without breaking the capsule. `nonos_manifest` also carries
the names-to-bits mapping (`cap_bit`, `cap_mask` in `src/caps/`) and the endpoint
model (`Endpoint`, `EndpointKind`), and `parse_document` reads the manifest
source into this structure. `build_sign_args` turns a manifest into the argument
list the signer expects, which is how the manifest and the signature stay
consistent.

## Sign

Signing goes through `nonos_sign_bridge`
(`userland/platform/nonos_sign_bridge/`): `sign_manifest` and `run_signer` invoke
the `capsule-sign` tool to produce the certificate and the signed manifest. The
crate is a bridge on purpose, so the CLI does not embed key handling; that lives
in the signer. What comes out is a manifest plus certificate that the kernel's
trust anchor can verify.

## Package

`nonos_package` (`userland/platform/nonos_package/`) is the bundle format:
`package_dir` collects a built capsule directory into a `Package`
(`userland/platform/nonos_package/src/model.rs`)
ready to be installed or moved. It is deliberately thin; the interesting checks
are at install time, not packaging time.

## Install, with verification that refuses on failure

`nonos_install` (`userland/platform/nonos_install/`) is the gate. Before anything
is written, `verify` runs the signer's `verify-manifest` against the manifest,
certificate, and policy:

```rust
if !status.success() {
    return Err("manifest verification failed; refusing install".into());
}
```

That is the posture the whole system takes: a capsule whose signature or policy
does not check does not get installed, full stop. `verify_payload` extends the
same check to the binary itself so the thing being installed is the thing that
was signed. Only after both pass does `install` write the capsule and record it
in the registry, and `remove` reverses it.

## The registry

`nonos_registry` (`userland/platform/nonos_registry/`) is the record of what is
installed: `add`, `find`, `load`, `remove`, `save`, and the `RegistryEntry`
type, with its own `encode`/`decode`. It is what `nonos run` and the installer
consult to find an installed capsule by name, and what `inspect` reads to show
what a capsule declared without running it.

## Where this meets the kernel

Everything above is host-side tooling that produces a signed artifact. The
kernel side, which verifies the certificate against the trust anchor, checks the
manifest, and attests the capsule at spawn, is documented in
[../../security/capsules-and-trust.md](../../security/capsules-and-trust.md) and
[../../security/manifest-schema.md](../../security/manifest-schema.md). The build
and signing commands are also covered from the Make side in
[../../build/signing.md](../../build/signing.md). This page is the developer's
path; those are the enforcement.

## Status

| Step | Library | Status |
|---|---|---|
| Command dispatch (`nonos <cmd>`) | `nonos_capsule/src/dispatch.rs` | IMPLEMENTED |
| Build via cargo + NONOS target | `nonos_capsule/src/cmd/build.rs` | IMPLEMENTED |
| Manifest model, parse, cap mapping | `nonos_manifest/src/` | IMPLEMENTED |
| Sign via `capsule-sign` | `nonos_sign_bridge/src/` | IMPLEMENTED |
| Package a capsule directory | `nonos_package/src/` | IMPLEMENTED |
| Install with signature + payload verify | `nonos_install/src/verify.rs`, `verify_payload.rs` | IMPLEMENTED; ENFORCED (refuses on failure) |
| Installed-capsule registry | `nonos_registry/src/` | IMPLEMENTED |

## Source

`userland/platform/`. Start at `nonos_capsule/src/dispatch.rs` for the command
surface, then follow one command through: `cmd/build.rs`, then `nonos_manifest`
for the format, `nonos_sign_bridge` for signing, and `nonos_install/src/verify.rs`
for the install gate.

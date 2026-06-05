# Capsule Signing

This page describes the capsule signing pipeline: `capsule-sign`,
`Capsule.mk`, the trust anchor policy, certificate and manifest artifacts, and
kernel-side verification. Read [Toolchain](toolchain.md) first.

---

## 1. Trust layout

The root Makefile defines trust material under `nonos-data/trust`, with trust
anchor public keys in `keys`, the sealed policy in `policy`, and private seeds
under `.keys` (`Makefile:257`). The sealed policy output is
`nonos-data/trust/policy/nonos_trust_anchor.policy.bin`
(`Makefile:271`). The host signing binary is
`nonos-sign/target/release/capsule-sign` (`Makefile:280`).

The trust policy rule depends on Ed25519 and ML-DSA-65 trust anchor public keys
and runs `capsule-sign mk-trust-policy` with epoch, both public keys, validity
window, and output path (`Makefile:333`).

## 2. Capsule metadata

The shared capsule macro requires every capsule to set slug, binary name,
directory, handle, domain, namespace, service endpoint, reply endpoint, and
required caps before including `nonos-mk/capsule.mk`
(`nonos-mk/capsule.mk:28`). Optional caps default to `0x0`, caps ceiling
defaults to required caps, target defaults to `x86_64-nonos-user`, version
defaults to `0.1.0`, and build std defaults to `core,alloc`
(`nonos-mk/capsule.mk:70`).

The macro writes each capsule binary path under its own target directory, the
NØNOS-ID certificate to `nonos-data/trust/capsules/<bin>.nonos_id_cert.bin`,
and the manifest to `nonos-data/trust/capsules/<bin>.manifest.bin`
(`nonos-mk/capsule.mk:91`).

## 3. Certificate rule

For each capsule, the macro derives `nonos_id` from handle, domain, and
recovery value using `capsule-sign derive-id` (`nonos-mk/capsule.mk:175`). The
certificate rule signs with serial, nonos id, namespace glob, caps ceiling,
trust anchor epoch, validity window, Ed25519 publisher key, ML-DSA-65 publisher
key, Ed25519 trust anchor seed, ML-DSA-65 trust anchor seed, metadata, and
output path (`nonos-mk/capsule.mk:180`).

## 4. Manifest rule

The manifest rule depends on the capsule ELF, certificate, Capsule.mk, and
signing tool. It runs `capsule-sign sign-manifest` with certificate, namespace,
version, target, ELF path, required caps, optional caps, service endpoint, reply
endpoint, Ed25519 publisher seed, ML-DSA-65 publisher seed, and output path
(`nonos-mk/capsule.mk:201`). The same rule verifies the manifest against the
certificate and trust policy before it is considered built
(`nonos-mk/capsule.mk:217`).

```
  +-------------------+
  | Capsule.mk        |
  | identity and caps |
  +---------+---------+
            |
  +-------------------+       +-------------------+
  | capsule ELF       |       | publisher keys    |
  +---------+---------+       +---------+---------+
            |                           |
            +-------------+-------------+
                          |
  +-------------------------------------+
  | capsule-sign sign-id-cert           |
  | capsule-sign sign-manifest          |
  | capsule-sign verify-manifest        |
  +----------------+--------------------+
                   |
  +-------------------------------------+
  | nonos-data/trust/capsules           |
  | <bin>.nonos_id_cert.bin             |
  | <bin>.manifest.bin                  |
  +-------------------------------------+
```

## 5. Kernel embedding and verification

Kernel mirror modules embed the signed outputs with `include_bytes`. The desktop
shell mirror embeds the capsule ELF, certificate, and manifest from their build
and trust locations (`src/userspace/capsule_desktop_shell/embed.rs:18`).

At spawn, the mirror decodes the baked trust anchor policy and builds a
`CapsuleSpecVerified` with embedded ELF, certificate, manifest, target triple,
service endpoint, reply endpoint, and requested caps
(`src/userspace/capsule_desktop_shell/spawn.rs:36`).

Preflight decodes and verifies the NØNOS-ID certificate, declares the service
and reply endpoints, and verifies the manifest with publisher keys, trust
anchor, ELF, target triple, requested caps, and declared endpoints
(`src/kernel_core/process_spawn/capsule_spawn/runner/preflight.rs:29`).
`spawn_verified` installs caps from the verified manifest result and passes that
to process install (`src/kernel_core/process_spawn/capsule_spawn/runner/verified.rs:26`).

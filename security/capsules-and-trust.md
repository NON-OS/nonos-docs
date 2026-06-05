# Capsules, Signing, and the Trust Anchor

This page covers how a capsule is identified, signed, and admitted to run. It is
the heart of the security model: no capsule executes until its certificate and
manifest verify against a trust anchor baked into the kernel, and its payload
hash matches the code being loaded. The architecture overview sketches this; here
is the full machinery with every structure and check.

Related: [capabilities and tokens](capabilities-and-tokens.md) for what a verified
capsule is then allowed to do, and [verified spawn](#verified-spawn) below for
the runtime gate.

---

## The three artifacts

A capsule is built ahead of time into three files:

```
  <bin>                              the ELF, compiled for x86_64-nonos-user
  <name>.nonos_id_cert.bin           the identity certificate, anchor-signed
  <name>.manifest.bin                the per-build manifest, publisher-signed
```

All three are compiled into the kernel with `include_bytes!`. The PS/2 input
driver is a representative example (`src/hardware/ps2_kbd_capsule/embed.rs:24`):

```
  DRIVER_PS2_INPUT_ELF                = include_bytes!(".../release/driver_ps2_input")
  DRIVER_PS2_INPUT_NONOS_ID_CERT_BYTES = include_bytes!(".../driver_ps2_input.nonos_id_cert.bin")
  DRIVER_PS2_INPUT_MANIFEST_BYTES     = include_bytes!(".../driver_ps2_input.manifest.bin")
```

The signed certificate and manifest live under `nonos-data/trust/capsules/`,
which is itself a tracked keystore submodule. There is no path where a capsule is
read from a mutable filesystem at runtime. The trust material is fixed at the
moment the kernel is built.

## The NONOS-ID certificate

The certificate is the durable identity of a capsule publisher and the ceiling
on what any build under that identity may ever do
(`src/security/nonos_id_cert/schema/cert.rs:25`, schema version 2):

```
  NonosIdCertificate
    schema_version           u16
    cert_serial              u64        identifies this certificate for revocation
    nonos_id                 [u8; 32]   BLAKE3-derived identity
    namespace_globs          Vec<..>    up to 8, the namespaces this id may serve
    allowed_caps_ceiling     u64        the maximum capabilities, ever
    metadata                 [u8; 256]  plus length
    valid_from_ms            u64
    valid_until_ms           u64
    trust_anchor_epoch       u64
    publisher_keys           Vec<..>    up to 4 publisher keys
    trust_anchor_signatures  Vec<..>    up to 4 anchor signatures
```

The `nonos_id` is derived deterministically by the signing tool as a BLAKE3 hash
over the capsule's immutable properties (handle, domain, recovery), so the
identity is reproducible and cannot be forged by editing a field. The
`allowed_caps_ceiling` is the hard limit: it is set when the certificate is
signed by the trust anchor, and no manifest under this identity can request more
than the ceiling allows. The certificate is signed by the trust anchor, so a
publisher cannot mint their own identity.

When verification succeeds, the certificate collapses to a small verified value
that the rest of the pipeline trusts:

```
  VerifiedNonosId
    nonos_id              [u8; 32]
    cert_serial           u64
    allowed_caps_ceiling  u64
```

## The manifest

The manifest is the statement a specific build makes about itself
(`src/security/capsule_manifest/schema/manifest.rs:28`, schema version 3):

```
  CapsuleManifest
    schema_version        u16
    nonos_id_cert_id      [u8; 32]   BLAKE3 of the certificate this binds to
    namespace             [u8; 96]   plus length
    version               Version
    target_triple         [u8; 64]   plus length
    payload_hash          [u8; 32]   BLAKE3 of the ELF
    required_caps         u64        bits that must be granted
    optional_caps         u64        bits that may be granted
    endpoints             Vec<EndpointDecl>        up to 16
    publisher_signatures  Vec<PublisherSignature>  up to 4
```

Two fields tie the manifest to reality. `nonos_id_cert_id` binds it to one
certificate, so a manifest cannot be paired with a different identity.
`payload_hash` is the BLAKE3 of the ELF being loaded, so a manifest cannot be
paired with different code. Both are checked at spawn.

An endpoint declaration (`schema/endpoint.rs`) names a kind, a port, and a name:

```
  EndpointKind = Service | Reply
  EndpointDecl { kind, port: u32, name: [u8; 48], name_len }
```

The set declared here is checked against the endpoints the kernel is asked to
register, so a capsule cannot quietly open a port it never declared.

## Signatures

A publisher signature carries the algorithm, a key id, and the bytes
(`schema/publisher_sig.rs`):

```
  PublisherSignature { algorithm: AlgId, key_id: [u8; 16], sig: [..], sig_len }
```

Production capsules are signed twice over the same material, once classical and
once post-quantum:

```
  AlgId                  public key      signature
  ------------------     -----------     -----------
  Ed25519   (0x01)       32 bytes        64 bytes
  ML-DSA-65 (0x03)       1952 bytes      3309 bytes
```

ML-DSA-65 is the NIST FIPS 204 lattice signature. Requiring both means a future
break of one scheme does not by itself forge a capsule. Verification dispatches
on algorithm (`src/crypto/asymmetric/alg_id/verify.rs:23`): Ed25519 through the
in-tree ed25519 code, ML-DSA-65 through the post-quantum module. Both must pass.

## The trust anchor

The anchor is the root of trust, baked into the kernel image
(`src/security/nonos_trust_anchor/baked.rs`, `BAKED_TRUST_ANCHOR_POLICY`). Its
schema (`schema.rs`):

```
  NonosTrustAnchorPolicy
    schema_version              u16
    trust_anchor_epoch          u64
    keys                        Vec<TrustAnchorKey>   up to 4
    revoked_cert_serials        Vec<u64>              up to 256
    revoked_nonos_ids           Vec<[u8; 32]>         up to 64
    revoked_publisher_key_ids   Vec<[u8; 16]>         up to 256
    flags                       u32

  TrustAnchorKey { algorithm, pubkey, pubkey_len, valid_from_ms, valid_until_ms }
```

The anchor signs certificates. A capsule's certificate is verified against these
anchor keys. The policy also carries revocation lists at three granularities: a
single certificate by serial, an entire identity by `nonos_id`, or a publisher
key by id. Revoking at the anchor invalidates everything beneath it at the next
build. The `trust_anchor_epoch` lets the system reason about which generation of
the policy a certificate was signed under.

---

## Verified spawn

Spawning is a gate, not a load. The entry point is `spawn_verified`
(`src/kernel_core/process_spawn/capsule_spawn/runner/verified.rs:26`), which runs
preflight, then install. Nothing executes until preflight returns.

```
  spawn_verified(spec, trust_anchor, now_ms)
        |
        v
  preflight  ->  verify_with_publisher    security/capsule_manifest/verify/mod.rs:36
        |
        v
  install                                  .../runner/install/install.rs:30
```

Preflight performs the following checks in order. Each is a separate function so
a failure points at exactly which invariant broke:

```
  decode certificate, verify it against the baked trust anchor
  decode manifest
  cert_binding::check     manifest.nonos_id_cert_id matches the certificate
  namespace::check        manifest namespace is within the certificate globs
  caps::check_ceiling     required_caps are within allowed_caps_ceiling
  signed_region::compute  determine exactly which bytes the signatures cover
  dispatch::run (per alg) verify each required signature over the signed region
  payload::check          BLAKE3 of the ELF equals manifest.payload_hash
  target_triple::check    the build targets this machine
  endpoint_drift::check   declared endpoints match what is being registered
  caps::check_grant       intersect with what is actually being granted
```

The output is the set of capability bits to install: the intersection of what the
manifest requested and what the certificate ceiling allows. That set, not the
raw request, is what the capsule receives.

Install then constructs the process (`install.rs:30`):

```
  create_process(name, state = Ready, Priority::Normal)
  register the process inbox  proc.<pid>
  load_elf_into_pid           map ELF segments into the process address space
  install_caps                write the verified capability bits into the PCB
  allocate_kernel_stack
  allocate_user_stack
  setup_initial_user_context  build the iretq frame: entry RIP, user RSP
  register_endpoint           the capsule's declared service port
  add_to_run_queue            tail of the runqueue
```

The process is created `Ready`, not `Running`. It does not execute until the
scheduler reaches it and performs the first-entry context switch that drops to
ring 3 at the ELF entry point (see [the scheduler page](../subsystems/scheduler.md)).
The initial context builder rejects any entry point or user stack above the
canonical user boundary before it will construct the frame
(`src/arch/x86_64/context/setup.rs:36`), so a manifest cannot be crafted to start
a capsule executing in kernel space.

---

## What this buys

The properties that fall out of the pipeline above:

```
  identity cannot be forged       certificates are anchor-signed; nonos_id is a hash
  code cannot be swapped          payload_hash binds the manifest to the exact ELF
  privilege cannot be escalated   ceiling bounds the manifest; intersection bounds the grant
  endpoints cannot be smuggled    declared endpoints are checked against registration
  a break needs two schemes       Ed25519 and ML-DSA-65 both sign, both must verify
  revocation is global            anchor revocation lists invalidate certs, ids, and keys
```

None of this depends on a trusted disk, a secure boot chain external to the
kernel, or a network handshake. The trust is baked into the image and checked in
RAM before the first capsule instruction runs.

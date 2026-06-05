# Security

How NØNOS decides what code may run and what running code may do. The two pages
here are the two halves of one story: admission, then enforcement.

| Page | What it covers |
|------|----------------|
| [capsules-and-trust.md](capsules-and-trust.md) | The capsule format, the NONOS-ID certificate, the manifest, publisher signatures (Ed25519 and ML-DSA-65), the baked trust anchor, and the verified-spawn gate that runs every check before a capsule executes. |
| [capabilities-and-tokens.md](capabilities-and-tokens.md) | The 22 capability bits, how they are declared and bounded by the certificate ceiling, how they are enforced on every syscall, and the MAC-authenticated capability token that binds them to a session, an address space, and a boot. |

Read them in that order. The capability set a capsule receives is the output of
the verified-spawn pipeline, so admission comes first.

Sources behind this section live under `src/security/` (capsule manifest,
NONOS-ID certificate, trust anchor), `src/capabilities/` (bits and tokens), and
`src/crypto/` (the signature and hash primitives).

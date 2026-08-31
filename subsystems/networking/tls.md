# TLS 1.3

Everything NONOS sends to the outside world that claims to be private leaves
through this crate. The Nym directory fetch, the browser's HTTPS, the wallet
talking to a node: all of it rides `nonos_tls`. It is a TLS 1.3 client and
nothing else. There is no server, no 1.2 fallback, no renegotiation, no
grab-bag of options nobody uses. It speaks one version of one protocol, and it
speaks it the same way for every caller.

That last part is the reason the crate exists at all. The browser used to carry
its own TLS, the wallet carried another, and the wallet's copy had already
drifted from the browser's in twenty one files. A certificate check that only
half the system gets is worse than no shared code, because it looks like
coverage while leaving a hole. So the two copies were pulled into one and the
callers were pointed at it. When the issuer rule gets fixed now, it gets fixed
for everyone at once, which is the only kind of fix worth having in a trusted
path.

`src/lib.rs`

## The shape of the crate

Open the `src/` directory and the first thing you notice is that there are
eighty eight files and almost every one holds a single function. `nonce.rs`
builds a nonce. `cert_dns_match.rs` decides whether a certificate covers a host
name. `expand_label.rs` is HKDF-Expand-Label and nothing else. This is not an
accident or a lint gone feral. A TLS client is a pile of small, exact, mutually
suspicious steps, and when each step is its own file the diff that changes the
issuer rule touches `cert_is_ca.rs` and only `cert_is_ca.rs`. You can read one
step without the other eighty seven in your face, and you can point a test or a
proof at it by name. It reads oddly the first time and then you stop noticing.

The public surface is deliberately tiny. From `src/lib.rs`:

- `session::exchange` with `Io` and `SessionError` is the front door: hand it a
  transport and a host, get back a completed session or a reason it failed.
- `client_flight` builds the opening flight. `server_complete` and
  `server_finished_flight_ready` drive the reply side.
- `application_request`, `application_write`, `application_plaintext` are the
  post-handshake read and write path.
- `TrafficKeys` is the derived key material. `cert_is_ca` and `rtc_now` are
  exported on purpose so a caller that has to do its own certificate walk uses
  this crate's rules and this crate's clock, not a second copy of either.

Everything else is `mod`-private. A caller cannot reach in and assemble a
handshake by hand, which is the point.

## The handshake, flight by flight

TLS 1.3 folds the old multi-round dance into essentially one round trip, and the
crate follows that structure directly.

**Client flight.** `client_flight` (`src/client_flight.rs`, returning
`ClientFlight` from `src/flight.rs`) produces three things: the record bytes to
put on the wire, the handshake bytes to feed the transcript hash, and the
X25519 private key it just generated. The ClientHello it builds is not
negotiable in the usual sense. It offers one supported version (1.3, in
`src/ext_versions.rs`), one key-exchange group, its two cipher suites, its
signature algorithms (`src/ext_sigalgs.rs`), the server name (`src/ext_sni.rs`),
and a single key share (`src/ext_keyshare.rs`). The private half of that key
share is carried back out in `ClientFlight.private` because the key schedule is
going to need it the moment the server answers.

**Server flight.** The server's reply is read and split apart by
`src/server_hello.rs`, `src/scan_server_finished.rs`, and the `server_*` files.
In 1.3 almost all of it is already encrypted: after the ServerHello, the
EncryptedExtensions, the Certificate, the CertificateVerify, and the server's
Finished all arrive under the handshake traffic keys. So the client cannot even
read the certificate until it has run the key schedule, which is the next
section. `server_complete` consumes the decrypted flight, runs certificate
verification, and checks the server's Finished against the transcript.

**Client Finished and application data.** Once the server's Finished verifies,
`client_finished.rs` and `finished_verify.rs` produce the client's own Finished,
the handshake is done, and the connection switches to application traffic keys.
From there `application_write` seals outbound data and `application_request` /
`application_plaintext` open inbound records. `application_plaintext_cached`
exists for the caller that wants to hold decrypted bytes without asking twice.

## The key schedule

This is the heart of it and it lives in `src/schedule.rs`, `src/hkdf.rs`, and
`src/expand_label.rs`. TLS 1.3 derives every key from a chain of HKDF-Extract
and HKDF-Expand-Label steps, each one mixing in the running transcript hash so
that a tampered handshake produces different keys and fails closed.

`handshake_keys` (`src/schedule.rs`) walks the early part of that chain. It
starts from a zero salt and zero IKM, extracts the early secret, expands a
`derived` secret over the hash of the empty string (the `EMPTY_HASH` constant is
just SHA-256 of nothing, precomputed so no allocation is needed to reproduce
it), then extracts the handshake secret over the X25519 shared secret. From the
handshake secret it expands the client and server handshake traffic secrets over
the current transcript hash, and from each of those it derives a record key and
an IV. The application keys come later from the same pattern over the full
handshake transcript.

Two details worth knowing before you touch this file. The transcript is hashed
with SHA-256 (`src/hash_sha256.rs`; `src/hash_sha384.rs` exists for the P-384
certificate path). And the record key length is not fixed: AES-128-GCM wants a
sixteen byte key, ChaCha20-Poly1305 wants thirty two, and HKDF-Expand-Label has
to be told the length up front. The crate derives into a buffer and the record
layer reads only as many bytes as its negotiated cipher uses, which is why the
key derivation and the cipher choice have to agree or you get a silent garbage
key rather than an error.

## The record layer

`src/record.rs`, `src/record_seal.rs`, and `src/record_open.rs` are the frame
boundary. Outbound, a plaintext record gets its inner content type appended
(`src/inner_plain.rs`), the additional-authenticated-data header built
(`src/aad_frame.rs`), a nonce computed as the IV XOR the record sequence number
(`src/nonce.rs`), and the whole thing sealed under the current traffic key.
Inbound is the same in reverse: open under the key, strip the padding back to
the real content type, advance the sequence. The sequence number is never sent;
both sides count, and if they disagree the AEAD tag simply does not verify,
which is the desired behavior.

## Certificate verification

A handshake that completes against an unverified certificate is worse than no
TLS, so this is where the crate spends most of its files. The chain arrives as
DER, gets counted and split (`src/cert_count.rs`, `src/cert_at.rs`), and each
certificate is parsed through a small DER TLV reader (`src/der_tlv.rs`) into its
pieces: the to-be-signed body (`src/cert_tbs.rs`), the issuer and subject, the
validity window, the extensions (`src/cert_ext.rs`), and the subject public key
(`src/cert_spki.rs`, `src/spki_point.rs`).

`src/chain_walk.rs` is the spine. It walks from the leaf toward a trust anchor,
and at each link it checks three things: that the issuer of one certificate
matches the subject of the next (`src/cert_issuer.rs`), that the signature over
the TBS actually verifies under the issuer's key, and that intermediates are
allowed to be intermediates (`src/cert_is_ca.rs`, the CA basic-constraint that
the browser now shares instead of reimplementing). The leaf also has to cover
the host name being dialed (`src/cert_dns_match.rs`) and be valid at the current
time.

Time comes from `src/rtc_now.rs`, which is worth a pause. A client with no clock
cannot check certificate validity, and NONOS is amnesic and RAM-resident, so
"the clock" is whatever the RTC says this boot. That is honest but it means
certificate expiry checks are exactly as trustworthy as the machine's real-time
clock, no more.

Signatures are verified by `src/verify_p256.rs`, `src/verify_p384.rs`, and
`src/verify_rsa.rs`, selected by the certificate's signature algorithm
(`src/cert_sig_alg.rs`) with the raw ECDSA encoding handled in
`src/ecdsa_sig_raw.rs`. The trust anchors themselves are a small pinned set: the
chain does not walk to an arbitrary system store, it walks to roots this crate
carries, which is the right posture for a client that only ever talks to a known
handful of endpoints.

## The line between local and pool crypto

The crate does not implement bulk symmetric crypto or the heavy public-key math
itself. It hands that to the `crypto_pool` service over IPC.
`src/crypto_port.rs` looks the service up by name at
`mk_service_lookup("crypto_pool")` and every AEAD seal or open and every
signature verify becomes a message to that capsule and a reply back
(`src/crypto_status.rs`, `src/crypto_port.rs`).

This is a deliberate trust and code-sharing decision, and it has one operational
consequence you will meet in the logs. Early in boot the crypto pool is under
load from every capsule attesting at once, and a certificate-chain verify that
loses its turn can come back as a transient failure rather than a wrong answer.
Upstream, in the Nym directory fetch, that surfaces as the certificate step
failing with code 20 while the mix and gateway fetches on the same host
succeeded. It is contention, not a broken endpoint, which is why the directory
sync retries it rather than giving up. See [mixnet.md](mixnet.md) for how that
retry is now structured.

## What it does not do

Stated plainly so nobody builds on a capability that is not there:

- Client only. There is no server side.
- One key-exchange group and a single name-pinned trust posture. This is not a
  general-purpose TLS library and should not be pointed at arbitrary hosts.
- No session resumption, no 0-RTT, no HelloRetryRequest, no client
  authentication. A server that demands any of those will not complete.
- Certificate time checks are only as good as the RTC this boot.

## Status

| Mechanism | Source | Status |
|---|---|---|
| TLS 1.3 handshake, one round trip | `session.rs`, `client_flight.rs`, `server_complete.rs` | IMPLEMENTED; DEMONSTRATED under QEMU |
| Key schedule (HKDF, SHA-256 transcript) | `schedule.rs`, `hkdf.rs`, `expand_label.rs` | IMPLEMENTED |
| AES-128-GCM and ChaCha20-Poly1305 records | `record.rs`, `aes_gcm.rs`, via `crypto_pool` | IMPLEMENTED |
| Certificate chain walk to pinned roots | `chain_walk.rs`, `cert_*.rs`, `verify_*.rs` | IMPLEMENTED; ENFORCED (fail-closed) |
| Host name and validity checks | `cert_dns_match.rs`, `rtc_now.rs` | ENFORCED; time bounded by the RTC |
| Resumption / HRR / client auth | absent | NOT IMPLEMENTED |

The proven path is the Nym directory fetch and the browser under QEMU. There is
no real-hardware TLS interop test in the tree.

## Source

`userland/nonos_tls/src/`. Start at `lib.rs` for the public surface, then
`session.rs` for the front door, `flight.rs` and `client_flight.rs` for the
opening flight, `schedule.rs` for the key derivation, `record.rs` for framing,
and `chain_walk.rs` for certificate verification. The crypto pool boundary is
`crypto_port.rs`.

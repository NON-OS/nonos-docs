# Debugging capsule_net_dhcp

This page lists the boot marker the DHCP capsule's spawn path emits, the two startup loops, and the concrete
runtime failure modes with where to look for each. For the shape of the capsule see the [README](README.md),
the [operations](operations.md) page, the [lease](lease.md) page, the [transport](transport.md) page, and the
[framing](framing.md) page.

## The boot marker

The first thing to confirm is that the capsule ran. On a successful boot the kernel prints
`[NET-DHCP] capsule spawned`: the network spawn plan calls `boot::capsule` with the tag `NET-DHCP`
(`src/userspace/init/spawn_plan/network/spawn_dhcp.rs:21`, via `src/userspace/init/spawn_plan/boot.rs:20`),
whose `Ok` arm calls `boot_log::ok(prefix, "capsule spawned")` (`src/userspace/init/capsule_boot/run.rs:29`),
which prints `[` + tag + `] ` + message (`src/sys/boot_log/output.rs:33`). An absent line means the capsule
never started, usually a signature, manifest, or capability failure; the `Err` arm prints an `[ERROR]` line
instead (`src/userspace/init/capsule_boot/run.rs:32`, `src/sys/boot_log/output.rs:50`). The line only appears
at all when the kernel is built with the `nonos-capsule-net-dhcp` feature, the `cfg` gate on the spawn entry
(`src/userspace/init/spawn_plan/network/spawn_dhcp.rs:18`).

## Setup never completes (the capsule is spawned but silent)

Unlike a driver, this capsule does not exit on a setup failure. `_start` calls `setup::run` in a loop, yielding
64 times between attempts and retrying forever until it succeeds (`src/main.rs:44`). So a capsule that spawned
but never answers a request is almost always stuck in that loop because `net.l2` or `net.ip` has not come up.
`setup::run` fails for one of three reasons (`src/setup/run.rs:45`):

- `mk_service_lookup("net.l2")` returned non-zero or a zero port, so the L2 capsule is not registered yet
  (`SetupError::L2NotFound`, `src/setup/run.rs:47`, `src/setup/discover.rs:28`). This is the expected transient
  at boot, and the retry loop is there to wait it out.
- The L2 lookup succeeded but `net.ip` did not resolve (`IpNotFound`, `src/setup/run.rs:49`). Both service
  capsules have to be spawned and registered before DHCP can bind to them.
- Both lookups succeeded but `read_mac` failed, so `net.l2` answered but the MAC read did not return six bytes
  (`L2MacFailed`, `src/setup/run.rs:52`, `src/l2_client/mac.rs:46`). That points at the L2 capsule or the NIC
  driver below it.

Because the loop retries silently, confirm `net.l2` and `net.ip` are up and that `net.l2` returns a MAC. Once
they do, the next setup attempt caches both ports and the MAC and the capsule proceeds to its initial acquire.

## Initial acquire fails (setup done, no lease)

After setup, `_start` runs the DORA ladder up to sixteen times, yielding 256 times between attempts, before it
gives up and starts the server anyway (`src/main.rs:55`). A capsule that answers `OP_LEASE_STATUS` with state
code 0 (Init) and an all-zero lease got no lease at boot, usually because no DHCP server answered under the
poll budget. A client can retry with `OP_LEASE_REQUEST`, which runs the same ladder synchronously and returns
the errno.

## Runtime failure modes

After the server is running, failures surface as errno words in the response header, not exit codes. Every
errno is the 16-bit value at offset 8 of the reply (`src/server/respond.rs:37`).

### A request gets no reply at all

Two request-side conditions produce no reply rather than an errno. A frame that fails to parse (wrong magic,
wrong version, or a length that does not match the declared body) is dropped in the request loop
(`src/server/runner.rs:43`), and a receive with a zero sender pid is skipped (`runner.rs:39`). If a client sees
a silent timeout rather than an errno, the request never parsed or carried no attributable sender. Confirm the
magic is `NDHC` (`0x4E44_4843`), the version field is `1`, and the `payload_len` field matches the body
actually sent (`src/server/parse_req.rs:27`).

### `E_BAD_OP` (3)

The opcode was not one of the five the dispatch handles (`src/server/runner.rs:50`). Only `1` through `5` are
defined (`src/protocol/ops.rs:17`).

### `E_NO_LINK` (5)

The capsule could not reach the wire or install the lease. On acquire and renew, it fires when the cached L2 or
IP port or the MAC is unset, meaning setup has not completed (`src/dora/acquire.rs:32`,
`src/server/handlers/xid_mac.rs:22`); when the transaction-id draw fails (`acquire.rs:35`,
`src/state/global.rs:56`); when a BOOTP send fails at L2 (`src/dora/send_bootp.rs:49`); or when the lease
install into `net.ip` is refused (`src/dora/install.rs:33`). On renew it also fires when no lease is currently
bound (`src/server/handlers/lease_renew.rs:39`). The first cases point back at setup, `net.l2`, and the NIC;
the install case at `net.ip`.

### `E_TIMEOUT` (6)

The DISCOVER or REQUEST wait ran out of its poll budget without a matching reply. `wait_for` polls `net.l2` up
to 4000 times, dropping non-matching frames, and returns `Timeout` if none matched (`src/dora/wait_reply.rs:42`).
This is the normal "no DHCP server answered" outcome under a dead or slow network; the client can retry
`OP_LEASE_REQUEST`.

### `E_NAK` (7)

The server answered a `DHCPREQUEST` with a `DHCPNAK`, refusing the requested address. On acquire, the ladder
resets client state to Init and returns `Nak` (`src/dora/request.rs:43`, `src/dora/acquire.rs:70`). On renew,
the same NAK collapses the state to Init, and the client should follow up with a fresh `OP_LEASE_REQUEST`
(`src/server/handlers/lease_renew.rs:56`). An ACK whose `yiaddr` is all zeros is also treated as a rejection
and surfaces as `E_NAK` (`src/dora/acquire.rs:41`).

### Release always succeeds

`OP_LEASE_RELEASE` sends a `DHCPRELEASE` and clears `net.ip` only if a lease is bound, then resets state, and
replies `E_OK` whether or not a lease was held; a release with no lease is idempotent
(`src/server/handlers/lease_release.rs:29`). It returns `E_NO_LINK` only when setup has not completed
(`lease_release.rs:33`).

## Source map

```
  src/userspace/init/spawn_plan/network/spawn_dhcp.rs   the NET-DHCP spawn entry and its cfg gate
  src/userspace/init/spawn_plan/boot.rs                 boot::capsule
  src/userspace/init/capsule_boot/run.rs                the capsule-spawned / error boot markers
  src/sys/boot_log/output.rs                            the [TAG] message formatting
  userland/capsule_net_dhcp/src/main.rs                 the setup retry and initial-acquire loops
  userland/capsule_net_dhcp/src/setup/run.rs            L2NotFound / IpNotFound / L2MacFailed
  userland/capsule_net_dhcp/src/server/parse_req.rs     the parse failures behind a dropped request
  userland/capsule_net_dhcp/src/server/runner.rs        the dispatch and the sender-pid gate
  userland/capsule_net_dhcp/src/protocol/errno.rs       the errno constants
  userland/capsule_net_dhcp/src/dora/acquire.rs, request.rs, wait_reply.rs, install.rs   the E_NO_LINK/E_TIMEOUT/E_NAK paths
  userland/capsule_net_dhcp/src/server/handlers/lease_renew.rs, lease_release.rs, xid_mac.rs   the renew/release paths
```

Every reference above is verified against those trees.

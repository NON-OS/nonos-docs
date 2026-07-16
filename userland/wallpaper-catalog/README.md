# capsule_wallpaper_catalog

The wallpaper catalog is the read-only asset vendor for the built-in wallpaper images. It reports how
many images there are, each image's slug and byte size, and streams the image bytes back in bounded
chunks so a two-megabyte JPEG never has to travel in one oversized message. It holds no per-caller
state and touches no hardware: every image is compiled into the capsule binary, so the catalog owns its
data and needs no storage device.

The capsule registers the service `wallpaper_catalog` on port 4110
(`userland/capsule_wallpaper_catalog/src/bootstrap/service_name.rs:17`,
`userland/capsule_wallpaper_catalog/src/bootstrap/port.rs:17`). Its capability mask is `0x19`, which is
`CoreExec | IPC | Memory` (`userland/capsule_wallpaper_catalog/Capsule.mk:14`). It requests no PCI,
MMIO, IRQ, DMA, PIO, filesystem, network, display, or focus-routing grants. The [wallpaper](../wallpaper/README.md)
capsule is the primary client; the [desktop shell](../desktop-shell/README.md) drives selection through the
[policy store](../policy/README.md).

## Startup

`_start` (`userland/capsule_wallpaper_catalog/src/main.rs:30`) initializes the heap, then calls
`bootstrap::register` (`register.rs:21`), which invokes `mk_service_register` with the name, length, and
port. A heap failure exits with code 1; a failed registration exits with code 2. On success it enters
`server::run(SERVICE_PORT)` (`main.rs:37`), which never returns.

## The server loop

`run` (`userland/capsule_wallpaper_catalog/src/server/runner.rs:24`) allocates one stack buffer of
`IPC_PAYLOAD_MAX` bytes (`limits.rs:18`, `4096 + 32 = 4128`) and loops. Each pass:

- `recv::poll` (`recv.rs:21`) calls `mk_ipc_recv_from` with a zero flag, so the receive is non-blocking.
  If it returns nothing, the loop yields with `mk_yield` and retries (`runner.rs:29`).
- A message shorter than the 16-byte header (`HDR_LEN`, `hdr.rs:17`) is dropped silently
  (`runner.rs:33`).
- `Header::decode` (`hdr.rs:37`) parses the fixed header. There is no magic number and no version field;
  the header is the compact catalog form.
- The `op` field selects a handler (`runner.rs:40`). Anything unrecognized is answered with `E_INVAL`
  (`runner.rs:45`).

The header is five little-endian fields in this order (`hdr.rs:29`):

```
  op          u16   operation code
  status      u16   E_OK on success, an errno on failure
  index       u32   wallpaper index for size/chunk/slug
  offset      u32   byte offset into the image for a chunk
  payload_len u32   number of body bytes that follow the header
```

## The operations

Four operation codes (`userland/capsule_wallpaper_catalog/src/protocol/ops.rs:17`):

```
  OP_GET_COUNT = 0x0001
  OP_GET_SIZE  = 0x0002
  OP_GET_CHUNK = 0x0003
  OP_GET_SLUG  = 0x0004
```

| Operation | Input | Reply body |
|---|---|---|
| `OP_GET_COUNT` | none | image count as a little-endian `u32` |
| `OP_GET_SIZE` | `index` | image byte length as a little-endian `u32` |
| `OP_GET_SLUG` | `index` | the image's slug bytes, no terminator |
| `OP_GET_CHUNK` | `index`, `offset` | up to 4096 bytes of the image starting at `offset` |

`OP_GET_COUNT` (`handlers/op_get_count.rs:22`) always replies with the total catalog count in `index=0`,
`offset=0`, and a four-byte body.

`OP_GET_SIZE` (`handlers/op_get_size.rs:22`) looks up the image bytes and returns their length; a missing
index replies `E_NOT_FOUND`. The size is derived, not stored: `get_size` is `get_bytes(index).map(len)`
(`catalog/get_size.rs:19`).

`OP_GET_SLUG` (`handlers/op_get_slug.rs:22`) returns the slug bytes for the index, or `E_NOT_FOUND`.

`OP_GET_CHUNK` (`handlers/op_get_chunk.rs:22`) is the streaming path. It fetches the image bytes, then:

- a missing index replies `E_NOT_FOUND` (`op_get_chunk.rs:25`);
- an `offset` strictly greater than the image length replies `E_RANGE` (`op_get_chunk.rs:28`); note that
  `offset == len` is allowed and returns an empty slice, which is how a client detects end-of-image;
- otherwise it returns `min(offset + CHUNK_MAX, len) - offset` bytes, at most 4096 (`op_get_chunk.rs:31`).

That is why a client fetches a large image incrementally: call `OP_GET_SIZE` once, then loop
`OP_GET_CHUNK` advancing `offset` by the returned `payload_len` until it reaches the size. The wallpaper
client's `fetch_image` (`userland/capsule_wallpaper/src/catalog_client/fetch_image.rs:26`) does exactly
this, capping any single image at 2,000,000 bytes and bounding the loop with a chunk count.

## Reply framing

Success replies go through `respond::ok` (`server/respond/ok.rs:24`). It rejects a payload larger than
`IPC_PAYLOAD_MAX - HDR_LEN` with `E_BAD_LEN` (`ok.rs:25`), then encodes the header with `status = E_OK`
and the true `payload_len`, copies the body in behind the header, and sends only the used prefix through
`reply::send` (`reply.rs:19`, which calls `mk_ipc_reply`).

Error replies go through `respond::err` (`server/respond/err.rs:21`). An error frame echoes the original
`op` and `index`, sets `offset = 0` and `payload_len = 0`, and carries only the 16-byte header. The
errno set (`protocol/errno.rs:17`) is:

```
  E_OK        = 0
  E_INVAL     = 22   unknown operation
  E_BAD_LEN   = 90   reply payload would exceed the buffer
  E_NOT_FOUND = 91   index has no image
  E_RANGE     = 93   chunk offset past the end of the image
```

The capsule's own `README.md` in the source tree names an `E_BAD_OP` code for unknown operations; that
name does not exist in `errno.rs`. The code actually returns `E_INVAL` (22) for an unknown op and does
not decode a body length separately, so there is no `E_INVAL`-versus-`E_BAD_OP` distinction in practice.

## The catalog

The catalog is not loaded at runtime. It is four static slices of `Entry`, each an embedded slug and an
`include_bytes!` of a JPEG under `nonos-data/wallpapers/`. `Entry` is just the slug and the byte slice
(`catalog/entry.rs:17`). The four groups (`catalog/entries/groups.rs:23`) are concatenated in this order:

| Group | Const | Entries | Slug pattern |
|---|---|---|---|
| Field focus | `FIELD_FOCUS` | 13 | `field-focus-1` .. `field-focus-13` |
| Hardware aesthetic | `HARDWARE_AESTHETIC` | 14 | `hardware-aesthetic-1` .. `hardware-aesthetic-14` |
| Network topology | `NETWORK_TOPOLOGY` | 18 | `network-topology-1` .. `network-topology-19`, skipping 12 |
| Special variant | `SPECIAL_VARIANT` | 17 | `special-variant-1a`, `-1b`, `-2a`, `-2b`, `-3` .. `-15` |

That is 62 served images. Note two details that make the file-on-disk count differ from the served
count. The `network-topology` group has no `network-topology-12` entry: it jumps from 11 to 13
(`catalog/entries/network_topology.rs:30`), so eighteen slugs span the numbers 1 through 19. And the
`special-variant-6` slug embeds `special-variant-6-1080p.jpg`, not `special-variant-6.jpg`
(`catalog/entries/special_variant.rs:27`); the plain `special-variant-6.jpg` file exists in
`nonos-data/wallpapers/` but is not referenced by any entry. There are 63 JPEGs on disk and 62 in the
catalog. The Capsule.mk comment that says "63 Full-HD wallpaper JPEGs"
(`userland/capsule_wallpaper_catalog/Capsule.mk:2`) counts disk files, not served entries.

Indexing is flat across groups. `entry_at` (`catalog/entries/entry_at.rs:20`) walks the groups in order,
subtracting each group's length until the remaining index lands inside a group, so index 0 is
`field-focus-1`, index 13 is `hardware-aesthetic-1`, and so on. `count` (`catalog/count.rs:19`) is the
saturating sum of the group lengths. Both are recomputed on every call; there is no cached total.

## How selection works

The catalog itself is stateless: it never picks a wallpaper, it only serves the image a caller asks for
by index. Selection lives in the [policy store](../policy/README.md). The store keeps a single `wallpaper: u8`
field (`userland/capsule_policy/src/store/types.rs:49`), and its default value is 52
(`userland/capsule_policy/src/store/defaults/store.rs:45`). That default is a flat catalog index: 52
falls inside the `special-variant` group, since 13 + 14 + 18 = 45 entries precede it.

The [wallpaper](../wallpaper/README.md) capsule reads that field over IPC. `get_wallpaper`
(`userland/capsule_wallpaper/src/policy_client/get_wallpaper.rs:22`) sends an `OP_GET` for
`Field::Wallpaper` to the policy service and returns the `u8`, which it then passes straight to
`fetch_image` as the catalog index. When a user changes the wallpaper in the settings panel, the panel
writes a new value into that same policy field, and the wallpaper capsule re-reads it and re-fetches.

The [desktop shell](../desktop-shell/README.md) also nudges the wallpaper capsule at prime time through
`apply_wallpaper_policy` (`userland/capsule_desktop_shell/src/setup/prime/run/apply_wallpaper_policy.rs:21`),
which calls `wallpaper_client::queue_policy(port, 3, 0)`. Those two arguments are a request id of 3 and a
policy code of 0 (`userland/capsule_desktop_shell/src/wallpaper_client/mod.rs:59`), not a catalog index.
The catalog index comes only from the policy store's `wallpaper` field.

## Security analysis

The catalog is a small read-only surface, and that is its main defense. It holds no secret, mutates no
state, and grants no authority: the capability mask is `CoreExec | IPC | Memory` and nothing else
(`Capsule.mk:14`). A compromised or malicious caller can request any image at any offset, but the images
are public background art, so there is nothing to leak.

Bounds are checked before every read. `entry_at` returns `None` for an out-of-range index rather than
indexing past the array (`entry_at.rs:20`), and the chunk handler rejects an offset past the end with
`E_RANGE` instead of clamping or wrapping (`op_get_chunk.rs:28`). The end computation uses
`core::cmp::min` so a large offset cannot produce an over-length slice (`op_get_chunk.rs:31`), and the
`ok` path refuses to frame a payload larger than the buffer (`ok.rs:25`). The receive buffer is fixed at
`IPC_PAYLOAD_MAX`, and `mk_ipc_recv_from` is bounded by that length (`recv.rs:21`), so an oversized
request cannot overflow it.

There is no per-caller authentication beyond the kernel capability check that lets a capsule reach the
service port at all. Any capsule with IPC rights can enumerate and download every image. That is
acceptable for public assets and is stated honestly in the source README's authority section. The
capsule never touches the sender pid except to address the reply (`reply::send`), so there is no
pid-based access decision to get wrong.

The receive loop is a poll-yield loop, not a blocking wait. A caller cannot pin the server on a slow
receive because the poll returns immediately; the only cost of a flood is repeated small replies, each
bounded to at most 4128 bytes.

## Debugging

If the desktop comes up with a blank or fallback background, walk the chain from the catalog outward.

- Confirm the service is registered. The name is `wallpaper_catalog` on port 4110
  (`service_name.rs:17`, `port.rs:17`). If registration failed, the capsule exited with code 2 before
  the loop (`main.rs:35`), so the service will not resolve through `mk_service_lookup` and the wallpaper
  client's `lookup_catalog` returns `None` (`userland/capsule_wallpaper/src/catalog_client/lookup.rs:23`).
- If the client gets a count but every image fails, suspect an index mismatch. The policy default is 52
  (`defaults/store.rs:45`); if the store was rebuilt with fewer than 53 entries the index would be out
  of range and `OP_GET_SIZE` would answer `E_NOT_FOUND`. Cross-check the served count with `OP_GET_COUNT`
  against the policy value.
- If a large image arrives truncated, the fault is usually in the chunk loop, not the server. The server
  returns exactly `min(offset + 4096, len) - offset` bytes and expects the client to advance `offset` by
  the reply's `payload_len` (`op_get_chunk.rs:31`). The client aborts if the running total ever exceeds
  the declared size or if the reassembled length does not match
  (`fetch_image.rs:43`, `fetch_image.rs:49`), so a truncated result means a dropped or reordered reply.
- An `E_RANGE` on a chunk means the client asked for an offset past the image end. An `E_NOT_FOUND` on
  size, slug, or chunk means the index is out of the 0..count range. An `E_INVAL` means the op code was
  not one of the four; check the header's `op` field encoding, remembering the header is little-endian
  (`hdr.rs:29`).
- To confirm which image an index maps to without decoding, request `OP_GET_SLUG` for that index and
  match the returned bytes against the group tables above.

## Source map

```
  userland/capsule_wallpaper_catalog/src/main.rs                       heap, register, run
  userland/capsule_wallpaper_catalog/src/bootstrap/                    service name, port 4110, register
  userland/capsule_wallpaper_catalog/src/server/runner.rs              poll/decode/dispatch loop
  userland/capsule_wallpaper_catalog/src/server/handlers/              count, size, chunk, slug
  userland/capsule_wallpaper_catalog/src/server/respond/               ok and err framing
  userland/capsule_wallpaper_catalog/src/protocol/                     header, ops, errnos, limits
  userland/capsule_wallpaper_catalog/src/catalog/                      count/size/slug/bytes accessors
  userland/capsule_wallpaper_catalog/src/catalog/entries/              the four embedded image groups
  userland/capsule_wallpaper_catalog/Capsule.mk                        caps 0x19, endpoints, asset deps
  userland/capsule_wallpaper/src/catalog_client/                       the client that streams images
  userland/capsule_wallpaper/src/policy_client/get_wallpaper.rs        reads the selected index
  userland/capsule_policy/src/store/types.rs                           the wallpaper field
  userland/capsule_policy/src/store/defaults/store.rs                  default index 52
  userland/capsule_desktop_shell/src/wallpaper_client/                 prime-time policy nudge
```

Every reference above is verified against those trees.

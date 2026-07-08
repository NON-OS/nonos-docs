# capsule_wallpaper_catalog

The wallpaper catalog serves the built-in wallpaper images: it reports how many there are, each one's
size and slug, and streams the image bytes in chunks so a large image need not be sent in one message.
It is a small, stateless catalog server. Service `wallpaper_catalog`, capability mask `0x19`. The source
is `userland/capsule_wallpaper_catalog/`.

## The server loop

`main.rs:30` initializes the heap, registers the service through bootstrap, and runs the loop
(`src/server/runner.rs:24`) over a 4 KiB buffer: poll for a request, decode the fixed 16-byte header
(op, status, index, offset, payload_len), dispatch, and reply. There is no magic or version; the header
is the compact catalog form.

## The operations

Four operations (`src/protocol/ops.rs:17`):

```
  GET_COUNT=1  GET_SIZE=2  GET_CHUNK=3  GET_SLUG=4
```

`GET_COUNT` returns the number of wallpapers; `GET_SIZE` takes an index and returns its dimensions and
byte total; `GET_SLUG` returns an image's name; `GET_CHUNK` (`src/server/handlers/op_get_chunk.rs`) takes
an index and an offset and returns up to 4 KiB of that image's bytes, which is how the
[wallpaper](wallpaper.md) capsule loads a large image incrementally rather than in one oversized
message.

## State and honesty

The catalog (`src/catalog/`) is an array of image metadata and embedded data, loaded at init from the
bootstrap resources, so the server is stateless per request and simply reads from the fixed catalog.
Honest scope: there is no pagination or filtering beyond the index, and no caller validation, the images
are public background art.

## Source

```
  userland/capsule_wallpaper_catalog/src/server/runner.rs     the poll/dispatch loop
  userland/capsule_wallpaper_catalog/src/server/handlers/      count, size, chunk, slug
  userland/capsule_wallpaper_catalog/src/catalog/              the embedded image catalog
```

# Capsule SDK

This page describes the userland SDK surface: `nonos_libc`, `nonos_abi`,
runtime crates, and the app skeleton used by GUI applications. Read
[Userland Model](README.md) first, then [Syscall ABI Reference](../abi/syscalls.md).

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

## 3. GUI app skeleton

`nonos_app_skeleton` is no_std and exports the app trait, manifest, input
types, paint buffer, clipboard helpers, and `run` entry point
(`userland/app_skeleton/src/lib.rs:17`). An app implements three methods:
`manifest`, `on_event`, and `paint` (`userland/app_skeleton/src/app/behavior.rs:21`).

The app manifest carries the window title, window id, window kind, initial
position, width, height, and input kind mask
(`userland/app_skeleton/src/app/manifest.rs:19`).

```
  App
    manifest()  -> title, window id, position, size, input mask
    on_event()  -> event outcome
    paint()     -> pixels in PaintBuffer
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

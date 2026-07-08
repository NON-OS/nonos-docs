# Userspace Init

Once the core is ready, the kernel creates one process, `init`, enters user space, and from there the
whole system is capsules. Init is the supervisor: it spawns the capsule fleet in dependency order and
then stays resident to watch over them. This page documents that transition. The code is
`src/kernel_core/init/entry.rs` and `src/userspace/init/`.

## Creating init

`microkernel_main` (`entry.rs:112`) is where the kernel crosses into user space:

```
  microkernel_main():
      spin-wait a short settling window (~2.5 s guard)
      init_pid = create_process("init", Running, High)
      create_address_space(init_pid)
      allocate_kernel_stack(init_pid)
      CURRENT_PID = init_pid
      userspace::run_init()
```

Init is created like any process, a [PCB](../process/pcb.md), a fresh [address space](../memory/unified-vm.md),
and a kernel stack, at high priority because it is the supervisor. Each of these steps is fatal on
failure: a system that cannot create its init process cannot boot, so the failure halts with a reason
rather than continuing. Once init is current, control enters `run_init`, which is the first code that
runs conceptually in user space.

## Spawning the fleet

`run_init` (`src/userspace/init/entry.rs`) spawns the capsule fleet in dependency order:

```
  run_init():
      spawn_ramfs()               the filesystem first
      spawn_core_after_ramfs()
      spawn_display_core()
      spawn_drivers()             bus, input, NIC drivers
      spawn_vfs()
      spawn_network()             net_core, sockets, nym
      spawn_desktop()             compositor, window manager, shell
      spawn_market()
      spawn_apps()
      lower_init_priority()
      init_loop()                 the supervisor loop
```

The order is a dependency order: the [filesystem](../storage/README.md) comes up first because later
capsules read from it, the [drivers](../hardware-broker/README.md) before the stacks that use them, the
[network](../networking/README.md) and [display](../graphics/README.md) before the desktop, and the
applications last. Each spawn goes through the [verified-spawn](../elf-loader/integration.md) pipeline,
so every capsule in the fleet is signature- and capability-checked as it is loaded. After the fleet is
up, init lowers its own priority so it does not compete with the capsules it launched.

## The supervisor loop

Init does not exit; it becomes the supervisor (`src/userspace/init/supervisor/`). The `init_loop` is
the resident parent that the [process lifecycle](../process/lifecycle.md) reaps children into: when a
capsule exits, init is where the final teardown accounting settles, and a supervised capsule that dies
can be restarted per policy. This is the userspace analogue of the kernel's fatal invariants, the
system always has a live supervisor, just as it always has a keyed IPC path and a working VM. From here
the system is running: the kernel holds the primitives, and the capsule fleet, filesystem, network,
display, and applications, does the work.

## Source

```
  src/kernel_core/init/entry.rs        microkernel_main, create init, enter userspace
  src/userspace/init/entry.rs           run_init, the fleet spawn order
  src/userspace/init/spawn_plan/        the per-group spawn plans
  src/userspace/init/supervisor/        the resident supervisor loop
```

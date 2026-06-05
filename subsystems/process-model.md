# Process Model

A capsule, once admitted, is a process: a process control block, an address
space, a capability token, and a place in the scheduler. This page covers the
PCB, the process table, the lifecycle, and the supervisor that watches over
every capsule. The [scheduler](scheduler.md) covers how a process is picked to
run; this page is what a process is.

---

## The process control block

Every process is a `ProcessControlBlock` (`src/process/core/pcb.rs:31`). It is a
rich structure; the fields that matter for the NØNOS model are these:

```
  identity
    pid                       the process id
    name                      a human label

  scheduling
    state                     Mutex<ProcessState>
    priority                  Mutex<Priority>   Idle, Low, Normal, High, RealTime

  address space
    cr3                       the page-table root for this process
    memory                    code range, virtual memory areas, resident pages
    mmap_va                   the mmap region tracker

  authority
    capability_token          RwLock<Arc<CapabilityToken>>
    caps_bits                 a cached bitmap of the held capabilities
    caps_manifest_installed   one-shot gate: a verified manifest installs once
    revocation_epoch          bumped on revoke to invalidate the token

  kernel and user transition
    kernel_stack_top          RSP0 for the TSS; 0 means no user mode expected
    pending_user_entry        the iretq frame for first entry to ring 3
    saved_user_context        user registers saved on preemption
    syscall_user_rsp          the user RSP captured on syscall entry

  ipc
    reply_inbox               the capsule's registered reply inbox name
```

The PCB also carries POSIX-shaped fields (file descriptors, signals,
credentials, scheduling statistics) for processes that need them, but the fields
above are what the capability microkernel turns on for a capsule. The
`capability_token` and `caps_bits` are the live authority the syscall resolver
checks ([capabilities and tokens](../security/capabilities-and-tokens.md)); the
`pending_user_entry` is what the first-entry context switch consumes to drop the
capsule into ring 3 ([boot](boot.md)).

## States

A process is in one of seven states (`src/process/core/types.rs:25`):

```
  New          created, not yet runnable
  Ready        runnable, on the runqueue
  Running      executing on a CPU
  Sleeping     waiting on a deadline or an event
  Stopped      halted by a debugger or a signal
  Zombie(c)    exited with code c, awaiting reap
  Terminated(c) reaped
```

The state is behind a `Mutex` on the PCB, so a transition is a locked write
(`*pcb.state.lock() = ProcessState::Terminated(code)`). A spawned capsule starts
`Ready`, becomes `Running` on its first-entry switch, cycles through `Running`,
`Ready`, and `Sleeping`, and ends `Terminated`.

---

## The process table

All PCBs live in one global table (`src/process/core/table/types.rs`):

```
  ProcessTable { inner: RwLock<Vec<Arc<ProcessControlBlock>>> }
  static PROCESS_TABLE
```

Lookups are by pid (`find_by_pid`), a linear scan returning an `Arc` so the PCB
stays alive while a caller holds it. The common pattern is `with_process(pid,
|pcb| ...)` (`src/process/api.rs:54`), which resolves the pid, runs a closure
with the PCB, and drops the `Arc` after. Holding an `Arc` rather than a raw
reference means a process cannot be freed out from under code that is working
with it.

Pids are allocated from `NEXT_PID`, which starts at 1
(`src/process/core/table/types.rs:89`). Pid 1 is init, and it is special: the
inheritance rule grants pid 1 the ambient capability set, CoreExec, IPC, and
Memory (`src/process/core/table/inherit.rs:78`). Every other process inherits
from its parent, capped at that ambient set, before verified spawn installs the
capsule's actual capabilities.

---

## Creating a process

`create_process(name, state, priority)`
(`src/process/core/table/create.rs:26`) builds a process:

```
  create_process(name, state, prio)
    allocate a pid from NEXT_PID
    take the parent pid from CURRENT_PID
    compute inherited capabilities (capped at the ambient set)
    build the PCB with all fields initialised
    allocate the address space          paging::create_address_space(pid)
    rebind the address space to the capabilities
    insert into PROCESS_TABLE
    return the pid
```

Allocating the address space (`src/process/address_space/lifecycle/setup.rs:26`)
asks the paging manager for a fresh page-table root, then stores the resulting
CR3 handle in the PCB. From that point the process has its own private lower half
and the shared kernel upper half ([memory](memory.md)).

This is the create step. For a capsule, it is wrapped by verified spawn, which
also loads the ELF, installs the verified capabilities, allocates the stacks,
builds the first-entry frame, registers the endpoint, and enqueues the process
([capsules and trust](../security/capsules-and-trust.md)).

---

## The supervisor

Pid 1, init, is the supervisor. After the kernel drops into it, `run_init`
(`src/userspace/init/entry.rs:20`) spawns the system capsules in a fixed order
and then becomes a watchdog:

```
  run_init()
    spawn ramfs, then the core services
    spawn the drivers
    spawn the vfs
    spawn the network stack
    spawn the desktop
    spawn the market
    run the smoke tests
    lower init's own priority to Low
    init_loop()
```

`init_loop` (`src/userspace/init/supervisor/loop_impl.rs:25`) runs for the life
of the system:

```
  init_loop()
    loop:
      if a second has passed since the last tick:
          services::lifecycle::tick()    walk the capsule liveness registry
      yield_now()
```

The lifecycle tick checks each registered capsule for liveness
(`src/services/lifecycle/registry.rs`); when a capsule exits, its pid is cleared
and the next IPC to it observes a dead endpoint. Init itself runs at `Low`
priority so it never competes with the capsules it supervises; it wakes once a
second to sweep liveness and otherwise yields the CPU.

---

## Services and endpoints

A capsule that offers a service registers an endpoint
(`src/services/registry.rs`):

```
  ServiceEndpoint { name, port, pid, caps_required }
  register_endpoint(name, port, pid, caps)
    refuse if the caller may not register
    refuse a duplicate name or port (Exists)
    refuse if the table is full (Full)
    otherwise record the endpoint
```

Registration is done at spawn for the capsule's declared service port, and the
declared endpoints are checked against the manifest, so a capsule cannot register
a service it did not declare. Clients resolve a name to an endpoint with
`lookup_service`, or a port with `lookup_port`. This registry is what turns the
named-inbox routing in [IPC](ipc.md) into a directory: a client asks for a
service by name and gets the endpoint that serves it.

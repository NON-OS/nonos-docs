# The Process Lifecycle

A process moves through a fixed set of states from the moment it is created to the
moment its memory is reclaimed. This page documents the states, how a process is
created in a runnable but not-yet-running state, how it runs and blocks, and how it
exits in two phases: a teardown that releases everything it held and marks it a
zombie, and a reap that frees and zeros its memory. The state enum is in
`src/process/core/types.rs`, creation is in the [verified-spawn
gate](../../security/capsules-and-trust.md), and exit is under `src/process/exit/`.

## The states

```
  ProcessState
    New              constructed, not yet runnable
    Ready            in the run queue, waiting for a CPU
    Running          executing on a CPU
    Sleeping         blocked on a wait, off the run queue
    Stopped          stopped, typically by a signal
    Zombie(code)     exited, resources released, awaiting reap
    Terminated(code) fully torn down
```

`Running`, `Ready`, and `Sleeping` are the three the [scheduler](../scheduler/README.md)
moves a process between during its life; `Zombie` and `Terminated` are the two ends
of exit. The `Zombie` and `Terminated` variants carry the exit code, so a parent
reaping a child reads the code from the state itself.

## Creation

A process is not born running. The [verified-spawn](../../security/capsules-and-trust.md)
install path creates the `ProcessControlBlock` in the `Ready` state, loads the ELF into
a fresh address space, installs exactly the verified capability bits, allocates the
kernel and user stacks, builds the initial user context as the PCB's
`pending_user_entry`, registers the capsule's endpoints, and adds the pid to the tail
of the run queue. Nothing about the process runs until the scheduler reaches it, and
the first time it does, the [context switch](context-switch.md) consumes
`pending_user_entry` to drop it to ring 3 at its ELF entry point. Creation therefore
produces a fully-formed, authority-bearing, runnable process that has executed no
instructions yet.

## Running and blocking

Once runnable, the process alternates between `Ready` and `Running` under the
scheduler: it is dispatched to a CPU, runs until it yields or is preempted, and goes
back on the run queue. When it waits, on an [IPC](../ipc/README.md) receive with nothing
queued, an IRQ, or an explicit sleep, it transitions to `Sleeping` and comes off the
run queue entirely, so a blocked capsule does not spin. A later delivery or a passed
deadline wakes it back to `Ready`. The [scheduler](../scheduler/README.md) page documents
those transitions and the sleep and wake machinery in full.

## Exit, phase one: teardown

Exit begins at `exit_and_yield` (`src/process/exit/exit_and_yield.rs:26`), and its doc
comment names every site that calls it: the `MkExit` syscall, the default action of a
kill signal, and the ring-3 fault handlers, in other words every place where the
capsule's user address space is gone and there is nowhere for an `iretq` to return to.
It tears the current process down and then yields forever, since the context it was
called from is dead:

```
  exit_and_yield(exit_code, by_signal) -> !:
      teardown(current_pid, exit_code, by_signal)
      loop: select_next_process and switch to it, else idle the CPU
```

`teardown` (`src/process/exit/teardown.rs:21`) does the release work and is idempotent,
returning immediately if the process is already a zombie or terminated:

```
  teardown(pid, exit_code, by_signal):
      if already Zombie or Terminated -> return
      release surfaces owned by the pid, and forget its attach mappings
      release every broker resource: devices, IRQs, DMA, PIO grants
      defer the kernel-stack release (cannot free the stack it is on)
      store the exit code, set state = Zombie(exit_code)
      remove the pid from the run queue and clear it as current
      clear its preemption ticks
      enqueue the pid for the reaper
```

The order matters for correctness. A dying capsule's [broker](../hardware-broker/README.md)
grants, its claimed devices, bound IRQs, DMA buffers, and port-IO grants, are all
released here, so a device a crashed driver held is returned rather than stranded. Its
[surfaces](../graphics/README.md) are released. The kernel stack cannot be freed while the
CPU is still executing on it, so its release is deferred. Only then is the process
marked `Zombie`, taken off the run queue, and enqueued for the reaper. After teardown
returns, `exit_and_yield` never comes back: it selects another process and switches to
it, or idles.

## Exit, phase two: reap

Teardown leaves the process a zombie with its resources released but its memory still
mapped. The reaper drains the pending list and reclaims that memory through
`address_space` release, which clears the process's VMAs and then calls
`cleanup_address_space(asid)` (`src/process/address_space/lifecycle/release.rs`). That
ASID-scoped teardown frees the leaf frames and the page tables through
`frame_alloc::deallocate_frame`, and every freed frame is zeroed by `zero_frame` on the
way out. This is the point at which a capsule's memory is actually scrubbed, and it is
why the [ZeroState guarantee](../memory/zeroization.md) is the composition of exit
returning the frames and the allocator zeroing them, rather than a dedicated exit-time
wipe. Once its memory is reclaimed and its exit code collected, the process's entry is
removed from the [process table](process-table.md).

## Direct termination

Separately from the two-phase exit, `terminate(code)` on the PCB
(`src/process/core/pcb.rs:193`) records the exit code and sets the state directly to
`Terminated(code)` in one step. This is the harder, immediate transition used where a
process must be marked dead without going through the graceful teardown, and it is the
state a reaped process ends in.

## Source

```
  src/process/core/types.rs                          the ProcessState enum
  src/process/exit/exit_and_yield.rs                  the exit entry point
  src/process/exit/teardown.rs                        the resource-release teardown
  src/process/exit/pending.rs                         the reaper's pending list
  src/process/address_space/lifecycle/release.rs      address-space reclaim
  src/kernel_core/process_spawn/capsule_spawn/        creation (verified spawn install)
```

# The Process Table

The process table holds every live process, a per-CPU current-pid tracks which
process is running on each CPU, and a small allocator hands out process ids. This
page documents the table and its queries, the per-CPU current process, PID
allocation, and the creation path that builds a `ProcessControlBlock` and registers
it. The code is under `src/process/core/table/`.

## The table

The table is a vector of reference-counted PCBs behind a reader-writer lock
(`src/process/core/table/types.rs:56`):

```
  ProcessTable { inner: RwLock<Vec<Arc<ProcessControlBlock>>> }
  static PROCESS_TABLE: ProcessTable = ...
```

Every process is an `Arc<ProcessControlBlock>`, so the table shares ownership rather
than holding the only copy, and a caller that looks a process up gets an `Arc` it can
hold across a lock drop. The queries are all linear scans under the read lock
(`types.rs:60`):

```
  add(pcb)                 push a new process
  find_by_pid(pid)         the PCB with that pid, if live
  get_all_processes()      a clone of the whole vector
  get_children_of(ppid)    every process whose parent is ppid
  has_children(pid)        whether any process has this pid as parent
  is_active_pid / is_active_name   membership tests
```

The structure is a plain vector, so lookups are `O(n)` in the number of live
processes. That is a deliberate fit for the workload: a NØNOS system runs on the order
of dozens of capsules, not thousands of processes, so a linear scan under a shared
read lock is simpler and contends less than a map would, and the read lock lets the
common case, many concurrent lookups, proceed in parallel.

## The current process, per CPU

Which process is running is not a single global; it is per-CPU
(`types.rs:25`):

```
  CurrentPid { slots: [AtomicU32; MAX_CPUS] }
  static CURRENT_PID: CurrentPid = ...
```

`load`, `store`, and `swap` all operate on `slots[cpu_id()]`, the slot for the CPU
making the call, so each core independently records the process it is currently
running. This is what makes "the current process" correct under SMP: two cores can run
two different processes at the same time, and each reads its own current pid from its
own slot with no coordination. The [scheduler](../scheduler/README.md) updates this slot on
every context switch, and `current_pid()` throughout the kernel reads it.

## PID allocation

New ids come from a monotonic counter with wraparound and a liveness check
(`types.rs:93`):

```
  allocate_tid():
      lock the PID allocation mutex
      loop:
          current = NEXT_PID
          NEXT_PID = (current >= u32::MAX - 1) ? 1 : current + 1
          pid = current == 0 ? 1 : current
          if pid is not an active pid -> return Some(pid)
          after 65536 attempts -> log exhaustion, return None
```

`NEXT_PID` starts at 1 and advances on each allocation, wrapping back to 1 rather than
overflowing. Because the space wraps, a candidate id could still belong to a live
process, so the allocator checks `is_active_pid` and skips it, and it gives up after a
bounded number of attempts rather than looping forever if the space is genuinely full.
The allocation is serialised by a dedicated mutex so two cores cannot hand out the same
id.

## Creating a process

`create_process` builds a PCB and registers it (`src/process/core/table/create.rs:26`):

```
  create_process_with_mem(name, state, prio, mem_kb):
      if name is empty -> Err
      pid = NEXT_PID.fetch_add(1)
      parent = current pid
      caps = compute_inherited_caps(pid, parent)
      pcb = build_pcb(pid, parent, name, state, prio, mem_kb/4, caps)
      address_space::lifecycle::allocate(pcb)
      caps::rebind_address_space(pcb)          re-mint the token with the real ASID
      PROCESS_TABLE.add(pcb)
```

The new process inherits its capabilities from its parent through
`compute_inherited_caps`, then `build_pcb` constructs the full control block, an address
space is allocated for it, and its capability token is re-minted so `subject_asid`
reflects the real address space rather than the zero the base mint left. That re-mint
returns an error if the boot session nonce is not yet set, so a process cannot be
created without it, the same fail-closed rule the [signing path](../../security/signing-and-mac.md)
enforces. Only after all of that does the PCB join the table. Note that this is the
generic process constructor; a verified capsule goes through the
[spawn pipeline](../../security/capsules-and-trust.md), which additionally swaps the
inherited token for the manifest-derived one through the one-shot `install_spawn` gate.

## The initial control block

`build_pcb` (`create.rs:76`) sets every field's initial value, and several of the
defaults are worth stating because they are security-relevant:

```
  token           minted for pid over the inherited caps (fail-closed on boot nonce)
  io_bitmap       [0xFF; 8192]     every port denied; a PIO grant clears bits to allow
  kernel_stack_top 0               no user mode until a kernel stack is allocated
  pending_user_entry / saved_user_context   None
  exit_signal     17               SIGCHLD to the parent on exit
  cpus_allowed    !0               all CPUs
  umask           0o022            root_dir and cwd "/"
  memory.next_va  0x0000_4000_0000  the base mmap hands out from
```

The io_bitmap default is the important one: it is all ones, which on x86 means every
port is denied, so a fresh process has no port-IO permission at all, and a
[PIO grant](../hardware-broker/README.md) works by clearing the specific bits it authorises.
`kernel_stack_top` starts zero, which the scheduler reads as "no user mode expected"
until a kernel stack is allocated, and the token is minted at construction so the PCB is
never live without authenticated authority.

## Threads versus processes

`spawn_thread` (`create.rs:58`) creates a schedulable thread inside the current process
rather than a new process. The thread inherits the parent's address space through
`address_space::lifecycle::inherit`, so it runs on the same CR3, joins the parent's
thread group by copying its `tgid`, and takes a caller-provided user entry point and
stack. The distinction that matters for teardown is stated in the source comment: a
thread has no VMAs of its own, so tearing it down frees nothing of the shared address
space, which belongs to the process and is only reclaimed when the last member exits.
Unlike a process, a thread is given its kernel stack and initial user context inline
and added straight to the run queue.

## Source

```
  src/process/core/table/types.rs    ProcessTable, CurrentPid, PID allocation
  src/process/core/table/create.rs   create_process, spawn_thread, build_pcb
  src/process/core/table/inherit.rs  compute_inherited_caps
  src/process/core/table/ops.rs      the table operation wrappers
```

# Inboxes

The substrate under every message a capsule receives is the inbox: a named, bounded queue
with an explicit owner. A capsule does not share memory with another capsule and cannot
name another capsule's address space; the only way one reaches another is to enqueue a
message on a registered inbox, and the kernel decides whether that enqueue is allowed. This
page documents the inbox itself, the registry that owns them, the fail-closed enqueue, and
the lifecycle tie to process teardown. The code is under `src/ipc/nonos_inbox/`.

## The inbox

An `Inbox` (`src/ipc/nonos_inbox/inbox.rs:32`) is a mutex-guarded `VecDeque<IpcMessage>`
with a fixed capacity and an owner pid:

```
  struct Inbox {
      queue:    Mutex<VecDeque<IpcMessage>>,
      capacity: usize,
      owner:    u32,       // 0 = kernel-owned, else the capsule pid
      stats:    InboxStats,
  }
```

Enqueue is non-blocking and bounded: `try_enqueue` pushes only if `len < capacity`,
otherwise it records a drop and hands the message back (`inbox.rs:75`). There is no
unbounded growth and no blocking inside the lock; a full inbox is a fast, visible failure,
not a memory leak or a stall. Dequeue pops the front and records the dequeue. The queue is
FIFO, and the capacity is chosen at registration within fixed bounds,
`MIN_INBOX_CAPACITY = 16`, `DEFAULT_INBOX_CAPACITY = 1024`, `MAX_INBOX_CAPACITY = 65536`
(`registry.rs:39`).

## The registry

Inboxes live in one global registry, a `BTreeMap<String, Arc<Inbox>>` behind an `RwLock`
(`registry.rs:48`), keyed by name. Two names matter: `proc.<pid>`, the canonical
per-process inbox a capsule drains, and the kernel-owned reply inboxes the spawn pipeline
sets up. The registry's defining rule, stated in its own module doc, is that **there is no
auto-registration on the send or receive paths**. An inbox exists only because something
explicitly created it:

```
  register_inbox(name, owner_pid)            capsule-owned, fails if name taken
  register_or_get_bootstrap_inbox(name)      kernel-owned reply inbox, idempotent
```

`register_inbox` (`registry.rs:85`) rejects an empty name, rejects a capacity outside the
bounds, and rejects a name that is already registered, so a caller cannot silently take
over an existing queue. `register_or_get_bootstrap_inbox` (`registry.rs:118`) is the only
path that creates an inbox without a capsule pid; it stamps the owner as
`KERNEL_OWNER = 0` and is reserved for the reply inboxes the
[spawn pipeline](../process/lifecycle.md) pre-registers for the kernel to drain. It must
not be called from a normal send or receive.

## Fail-closed enqueue

Routing into an inbox goes through `try_enqueue_strict` (`registry.rs:164`), which fails
closed on three distinct conditions rather than papering over any of them:

```
  try_enqueue_strict(name, msg):
      inbox = registry.get(name)          else MissingInbox
      if owner != KERNEL_OWNER
         and process_table.find_by_pid(owner) is None:
              return DeadOwner
      inbox.try_enqueue(msg)              else QueueFull
```

`MissingInbox` means no such queue was ever registered. `DeadOwner` means the queue exists
but the capsule that owned it has fallen out of `PROCESS_TABLE`, which is what closes the
race where a destination exits between a caller's service lookup and its enqueue: the
kernel refuses to deliver into a dead capsule's queue. `QueueFull` means the bounded queue
is at capacity. Each maps to a distinct errno at the syscall layer, so the sender learns
which of the three happened. A kernel-owned inbox (`owner == 0`) skips the liveness check,
because its drainer is the kernel itself and never exits.

## Draining and the receive loop

A receiver drains its own inbox with `try_dequeue_existing`, which returns `None` on an
empty or absent queue and never creates one. The blocking behavior lives a layer up, in the
receive syscall (`src/syscall/microkernel/ipc/recv.rs:58`): it checks the inbox exists
(`ENOENT` if not), then loops, dequeue, and if empty either returns `ETIMEDOUT` past the
deadline or calls `sched::sleep_until` and re-checks on wake, yielding between spins. The
sender wakes a sleeping receiver explicitly after a successful enqueue, so a blocked
receiver does not spin against an empty queue. The [scheduler](../scheduler/sleep-wake.md)
sleep and wake are the mechanism; the inbox is the rendezvous.

## Lifecycle

When a capsule exits, `process::exit::teardown` calls `unregister_for_pid(pid)`
(`registry.rs:147`), which removes that capsule's `proc.<pid>` inbox and drops whatever was
still queued. The kernel-owned reply inboxes (`endpoint.<n>`) are deliberately left in
place so a respawn reuses them; stale replies are filtered by the transport's generation
re-check rather than by tearing the inbox down. This is why the `DeadOwner` check exists:
between a capsule exiting and a caller noticing, the strict enqueue is the backstop that
refuses delivery to the departed owner.

## Source

```
  src/ipc/nonos_inbox/inbox.rs      the bounded per-owner queue
  src/ipc/nonos_inbox/registry.rs   the global name -> inbox map, strict enqueue, lifecycle
  src/ipc/nonos_inbox/error.rs      InboxError and StrictEnqueueError
  src/syscall/microkernel/ipc/recv.rs   the blocking receive loop
```

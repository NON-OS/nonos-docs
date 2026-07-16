# The submission and completion queue engine

NVMe is a queue protocol. The driver writes a command into a submission queue in DMA memory, rings a
doorbell register to tell the controller the tail moved, and the controller writes a completion entry into a
completion queue and (optionally) raises an interrupt. This page mirrors the two folders that own that
machinery: `src/admin/` (the admin queue, its commands, and the Identify and SMART parsers) and `src/nvm/`
(the IO queue pair, the PRP path, and the read/write/flush transfers). How the queues are first allocated
and programmed is on the [bring-up](bring-up.md) page; the client ops that call into the IO queue are on the
[operations](operations.md) page.

## Two queue pairs

The driver runs two submission/completion pairs. The admin pair carries setup and management commands
(Identify, Create IO Queue, Get Log Page) and has 64 entries (`src/admin/queue/constants.rs:17`). The IO
pair carries block I/O (read, write, flush) for queue id 1 and has 8 entries
(`src/nvm/constants.rs:17`). Each pair is a submission queue and a completion queue in separate DMA
regions, plus the head and tail indices, a phase bit, and a rolling command id (`AdminQueue`, and `IoQueue`
which additionally holds its two doorbell offsets, the namespace id, and the namespace capacity in sectors).

## Command and completion layout

A submission entry is a 64-byte `#[repr(C, align(64))]` struct of sixteen dwords, and its size is asserted
at compile time (`src/admin/command/submission.rs:17`, `submission.rs:35`). A completion entry is a 16-byte
`#[repr(C, align(16))]` struct, also compile-time asserted (`src/admin/completion.rs:17`, `completion.rs:38`).
The completion's low status bit is the phase, and success is the rest of the status word being zero:

```
  phase()      = (status & 1) != 0          src/admin/completion.rs:29
  successful() = (status >> 1) == 0         src/admin/completion.rs:33
```

## Submit and reap

Submitting is the same shape on both queues. The driver writes the command into the SQ slot at the current
tail with a volatile write, advances the tail modulo the queue depth, and rings the SQ doorbell with the new
tail (`src/admin/queue/submit.rs:26`, `src/nvm/submit.rs:25`).

Reaping is a bounded poll. The driver reads the CQ slot at the current head and checks two things: the
completion's phase bit matches the queue's expected phase, and its command id matches the one just
submitted. A matched entry advances the head, flips the expected phase when the head wraps to zero, rings
the CQ doorbell, and returns success or `AdminCommandFailed` from the status word
(`src/admin/queue/wait.rs:27`, `src/nvm/wait.rs:26`). The phase-bit handshake is what lets the driver tell a
fresh completion from a stale slot without a valid flag: on every wrap of the queue the expected phase
flips, so a slot the controller has not written this lap still shows the old phase and is skipped. Each poll
loop spins up to a fixed limit (`5_000_000` iterations) and returns `ControllerTimeout` if the completion
never lands (`src/admin/queue/constants.rs:22`, `src/nvm/constants.rs:26`).

The command id rolls forward on every command and never becomes zero: `cid = cid.wrapping_add(1).max(1)`
after each submit (`src/nvm/transfer.rs:29`, `src/admin/queue/identify_controller.rs:26`).

## Doorbells and the stride

Doorbells live at and above `REG_DOORBELL_BASE` (`0x1000`), spaced by the controller's doorbell stride from
`CAP` (`src/constants/regs.rs:28`, `src/constants/cap.rs:25`). The admin doorbells are fixed: SQ0 tail is at
the base, and CQ0 head is one stride step up, computed as `REG_DOORBELL_BASE + (4 << stride)`
(`src/admin/queue/sq0_tail.rs:19`, `src/admin/queue/cq0_head.rs:19`). The IO doorbells are computed once at
allocation from the queue id: `REG_DOORBELL_BASE + (2 * qid) * stride_bytes` for the SQ tail and the next
step for the CQ head, where `stride_bytes = 4 << stride` (`src/nvm/alloc.rs:32`, `src/nvm/alloc.rs:42`).

## The admin commands

The admin queue drives one command per file under `src/admin/command/`, each a `const` `Submission`
constructor, submitted and waited by a matching `AdminQueue` method under `src/admin/queue/`.

| Command | Constructor | Queue method | Used by |
|---|---|---|---|
| Identify Controller (CNS 1) | `src/admin/command/identify_controller.rs:20` | `src/admin/queue/identify_controller.rs:24` | bring-up step 10 |
| Identify Namespace (CNS 0) | `src/admin/command/identify_namespace.rs:20` | `src/admin/queue/identify_namespace.rs:24` | bring-up step 11 |
| Get Log Page | `src/admin/command/get_log_page.rs:20` | `src/admin/queue/log.rs:27` | SMART snapshot |
| Create IO Completion Queue | `src/admin/command/create_io_cq.rs` | `src/admin/queue/create_cq.rs:23` | IO queue bring-up |
| Create IO Submission Queue | `src/admin/command/create_io_sq.rs` | `src/admin/queue/create_sq.rs:23` | IO queue bring-up |
| NVM Read/Write | `src/admin/command/nvm_rw.rs:20` | `src/nvm/transfer.rs:25` | block I/O |
| NVM Flush | `src/admin/command/nvm_flush.rs:20` | `src/nvm/flush.rs:23` | flush |

The Identify and Get Log Page commands all write into the admin queue's shared 4 KiB scratch DMA and hand
the parser back a slice of it (`src/admin/queue/identify_controller.rs:33`, `src/admin/queue/log.rs:42`).
The SMART Get Log Page uses LID `0x02`, NSID `0xffffffff`, and 512 bytes (`src/admin/queue/log.rs:22`).

## Identify and SMART parsers

The three cached records the driver serves are parsed once at bring-up from the raw controller data:

- `ControllerIdentity::parse` copies the 20-byte serial, 40-byte model, and 8-byte firmware and reads the
  vendor ids, version, optional-admin, namespace count, MDTS, SQ and CQ entry sizes, optional-NVM, and the
  volatile-write-cache byte at their fixed offsets (`src/admin/identity.rs:35`).
- `NamespaceIdentity::parse` reads size, capacity, and used in LBAs, decodes the active LBA format from the
  FLBAS nibble to derive the LBA size from its shift and the metadata size, and records the format index and
  formatted-LBA count; `absent()` is the all-zero record used when the controller reports no namespace
  (`src/admin/namespace.rs:43`, `src/admin/namespace.rs:30`).
- `SmartHealth::parse` reads the critical-warning byte, the composite temperature, spare, threshold,
  percentage-used, and endurance-group warning, then ten 128-bit lifetime counters and two 32-bit
  temperature-time counters at their log-page offsets (`src/admin/health/parse.rs:23`). The little-endian
  16/32/128-bit readers are one helper per file under `src/admin/health/`.

## The IO transfer path and PRP

A block transfer is `IoQueue::transfer`: it computes the byte count, builds the PRP pointers, allocates a
command id, submits an NVM read or write command, and waits the completion (`src/nvm/transfer.rs:25`). The
command carries the LBA in `cdw10`/`cdw11`, the zero-based block count `nlb = sectors - 1` in `cdw12`, and
the two PRP pointers, with opcode `0x01` for write and `0x02` for read
(`src/admin/command/nvm_rw.rs:20`). Flush is the same shape with opcode `0x00` and no data pointers
(`src/nvm/flush.rs:23`, `src/admin/command/nvm_flush.rs:20`).

PRP (Physical Region Page) is how NVMe names the data buffer. `build_prp` covers three cases against the
data DMA region's device address (`src/nvm/prp.rs:20`):

```
  bytes <= 4 KiB       -> (prp1 = data_phys, prp2 = 0)
  bytes <= 8 KiB       -> (prp1 = data_phys, prp2 = data_phys + 4 KiB)
  bytes  > 8 KiB       -> prp1 = data_phys; fill the PRP-list DMA with the
                          remaining page addresses; prp2 = prp_list_phys
```

The PRP list is written into its own 4 KiB DMA region, one 8-byte device address per remaining page
(`src/nvm/prp.rs:29`). Because the largest transfer is bounded, the list never overflows: `MAX_SECTORS` is
64, which is 32 KiB, which is eight pages and matches the data DMA region size
(`src/nvm/constants.rs:23`). The read handler copies the fetched bytes out of the data region into the reply
and the write handler copies the request bytes into the data region before submitting
(`src/server/handlers/read.rs:47`, `src/server/handlers/write.rs:36`).

## Where the interrupt fits

The interrupt is a wake hint, not the correctness mechanism. The server loop polls the MSI-X grant on every
iteration with `mk_irq_poll`, and when the sequence advances it acknowledges with `mk_irq_ack`
(`src/server/runner.rs:75`). The queue engine above never waits on the interrupt: every completion is found
by the phase-and-cid poll, which is why a missing MSI-X bind is not fatal and the driver runs correctly in
pure polling mode (`src/setup/irq.rs:24`).

## Source map

```
  userland/capsule_driver_nvme/src/admin/completion.rs    the 16-byte completion, phase() and successful()
  userland/capsule_driver_nvme/src/admin/command/         the Submission struct and one command builder per file
  userland/capsule_driver_nvme/src/admin/queue/           AdminQueue: allocate, program, submit, wait, doorbells
  userland/capsule_driver_nvme/src/admin/identity.rs      ControllerIdentity::parse
  userland/capsule_driver_nvme/src/admin/namespace.rs     NamespaceIdentity::parse and absent()
  userland/capsule_driver_nvme/src/admin/health/          SmartHealth::parse and the little-endian readers
  userland/capsule_driver_nvme/src/nvm/queue.rs           the IoQueue struct
  userland/capsule_driver_nvme/src/nvm/alloc.rs           IoQueue DMA regions and the computed doorbell offsets
  userland/capsule_driver_nvme/src/nvm/setup.rs           bring_up: create the IO cq then sq
  userland/capsule_driver_nvme/src/nvm/prp.rs             build_prp: prp1/prp2 and the PRP list
  userland/capsule_driver_nvme/src/nvm/transfer.rs        the read/write command path
  userland/capsule_driver_nvme/src/nvm/flush.rs           the flush command path
  userland/capsule_driver_nvme/src/nvm/submit.rs          IO submit and the SQ doorbell
  userland/capsule_driver_nvme/src/nvm/wait.rs            IO reap, phase flip, and the CQ doorbell
  userland/capsule_driver_nvme/src/nvm/constants.rs       IO_ENTRIES, MAX_SECTORS, SECTOR_SIZE, the poll limit
  userland/capsule_driver_nvme/src/constants/regs.rs      REG_DOORBELL_BASE and the CC encoding
  userland/capsule_driver_nvme/src/constants/cap.rs       the doorbell-stride decoder
```

Every reference above is verified against those trees.
</content>

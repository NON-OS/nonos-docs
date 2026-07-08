# Pipes

The inbox path carries discrete messages between capsules. The pipe is the other IPC
primitive: an anonymous byte FIFO behind a pair of file descriptors, the substrate for the
POSIX-shaped pipe the filesystem and file-descriptor table expose. It is a byte stream, not
a message queue, and it does not carry a sender identity or a MAC; it is a buffer with a
read end and a write end. The code is under `src/ipc/pipe/`.

## The buffer

A `Pipe` (`src/ipc/pipe/types.rs:26`) is a fixed-capacity ring buffer with independent read
and write positions and separate closed and non-blocking flags for each end:

```
  struct Pipe {
      buffer:          Vec<u8>,    // capacity bytes, allocated once
      read_pos, write_pos: usize,  // ring cursors
      bytes_available: usize,
      capacity:        usize,
      read_closed, write_closed:     bool,
      read_nonblock, write_nonblock: bool,
  }
```

The default capacity is `PIPE_BUF_SIZE = 65536` (`types.rs:20`), and the number of live
pipes is bounded by `MAX_PIPES = 1024`. The buffer is allocated at creation and never
grows; `space_available` is `capacity - bytes_available`, and the cursors wrap modulo the
capacity.

## Create, read, write

`create_pipe` (`src/ipc/pipe/api.rs:21`) allocates a pipe id, inserts the pipe into the
global table, and hands back two descriptors, a read fd and a write fd, each recorded in
the fd-to-pipe map with a flag marking which end it is
(`ENFILE`-style capacity error `24` if the table is full):

```
  create_pipe() -> (read_fd, write_fd)
```

`pipe_read` and `pipe_write` (`api.rs:43`, `api.rs:74`) look the fd up, enforce the end,
reading on a write fd or writing on a read fd is `EBADF`, and act on the ring:

- A read from an empty pipe returns `0` if the write end is closed (end of stream) and
  `EAGAIN` otherwise (`api.rs:58`). A non-empty read copies up to `min(buf, available)`
  bytes, advancing the read cursor.
- A write to a pipe whose read end is closed is `EPIPE` (`api.rs:89`); a write to a full
  pipe is `EAGAIN`. Otherwise it copies up to `min(buf, space)` bytes, advancing the write
  cursor.

Both are partial: they move as many bytes as fit and report the count, leaving the caller
to loop. The `is_broken` predicate (`types.rs:59`) is the end-of-stream condition, write
end closed and nothing left buffered.

## Where it is used

The pipe primitive is consumed by the filesystem and the process fd table
(`src/fs/pipe/`, `src/process/fd_table.rs`), which layer the blocking policy, the fd
lifecycle, and the syscall surface on top. Within IPC this module owns only the buffer and
the byte transfer; the message-passing path between capsules is the [inbox](inbox.md), not
the pipe. The two coexist: pipes give a capsule a stream fd it can hand to a child or wire
into a filter, and inboxes give the kernel a permission-checked, sender-attested message
channel.

## Source

```
  src/ipc/pipe/types.rs    the ring buffer, PIPE_BUF_SIZE, MAX_PIPES, error codes
  src/ipc/pipe/api.rs      create_pipe, pipe_read, pipe_write
  src/ipc/pipe/registry.rs the pipe and fd tables
  src/ipc/pipe/close.rs    pipe_close, non-blocking mode
```

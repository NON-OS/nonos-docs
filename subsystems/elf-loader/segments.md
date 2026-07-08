# Segment Loading and W^X

Each `PT_LOAD` segment becomes a run of mapped pages in the target address space. This is where
the loader enforces the invariant that no page is both writable and executable, derives page
permissions from the segment flags, and zero-fills backing memory so a segment never exposes
stale bytes or uninitialized `.bss`. This page documents `load_segment`. The code is under
`src/elf/loader/core/load_segment/`.

## The W^X gate

Before a segment is planned, `input_fields` (`src/elf/loader/core/load_segment/validate.rs:20`)
validates it, and the load-bearing check is the write-xor-execute gate:

```
  input_fields(elf_len, header):
      if p_filesz > p_memsz:                      SegmentDataOutOfBounds
      if header.is_writable() and header.is_executable():   WXViolation
      if p_align > 1 and not power_of_two(p_align):         AlignmentError
      if p_align > 1 and p_vaddr % p_align != p_offset % p_align:   AlignmentError
      if file_offset + file_size > elf_len:       SegmentDataOutOfBounds
```

A segment carrying both the writable and executable flags is rejected outright with
`WXViolation`. There is no code path that maps a `W|X` segment, so the classic attack surface of
a writable-executable region does not exist for a loaded capsule. The other checks reject a
segment whose file size exceeds its memory size, whose alignment is not a power of two, whose
virtual and file offsets are not congruent modulo the alignment (the ELF alignment invariant),
or whose file contents run past the end of the image.

## Permission derivation

For a segment that passes the gate, its page permissions are derived directly from its flags
(`load_segment/pte_flags.rs:20`):

```
  pte_perms_from_phdr(ph):
      perms = READ | USER          // always readable, always user
      if ph.is_writable():   perms |= WRITE
      if ph.is_executable(): perms |= EXECUTE
      perms
```

The base is `READ | USER`: a loaded capsule segment is always user-accessible and readable. Write
is added only for a writable segment and execute only for an executable one, so a read-only data
segment is mapped non-writable and non-executable, a text segment is read-execute, and a data
segment is read-write with no execute. Because the W^X gate already ran, `WRITE` and `EXECUTE`
are never both present. These permissions feed straight into the [paging manager](../memory/paging-manager.md),
which is the layer that actually enforces them in the page tables.

## Populating pages

`load_segment` (`load_segment/run.rs:28`) builds a page plan, derives the permissions once, and
populates each page (`load_segment/populate_page.rs`):

```
  populate_page(asid, page_va, perms, dst_off, src):
      frame = allocate_frame()                    else MemoryAllocationFailed
      if map_page_in_asid(asid, page_va, frame, perms) fails:
          free frame; MemoryAllocationFailed       // no leak on the error path
      dst = directmap(frame)
      write_bytes(dst, 0, PAGE)                    // zero the whole page first
      if src not empty:  copy src into dst at dst_off (clamped to the page)
```

Each page is allocated, mapped into the target address space with the segment's permissions, and
then zeroed in full through the direct map before the file bytes are copied in. Zeroing the whole
page first is what gives `.bss` for free: a segment whose `p_memsz` exceeds its `p_filesz` has its
trailing pages (and the tail of its last file page) already zero, with no separate `.bss`
handling. If the mapping fails, the just-allocated frame is freed before returning, so an error
midway through a segment does not leak physical memory. The pages are written through the kernel
direct map rather than through the user mapping, so the loader never dereferences a user virtual
address it just created.

## Source

```
  src/elf/loader/core/load_segment/validate.rs      the W^X gate and segment validation
  src/elf/loader/core/load_segment/pte_flags.rs      permission derivation from p_flags
  src/elf/loader/core/load_segment/run.rs            the per-page load loop
  src/elf/loader/core/load_segment/populate_page.rs  frame alloc, map, zero-fill, copy
```

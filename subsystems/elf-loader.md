# ELF Loader

A capsule's code is an ELF executable, verified by hash at spawn and then loaded
into the process address space by the ELF loader. This page covers what the
loader accepts, what it rejects, and how it maps a capsule into memory. The
[verified-spawn gate](../security/capsules-and-trust.md) runs first and confirms
the ELF's hash matches the manifest; the loader is what then turns those bytes
into a running image.

---

## Where it sits in spawn

Install calls `load_elf_into_pid`
(`src/kernel_core/process_spawn/capsule_spawn/runner/install/load_elf_into_pid.rs:21`):

```
  load_elf_into_pid(elf, pid, debug_tag)
    look up the address space id (asid) for the process
    load_elf_entry_into(elf, asid)
    return the entry point as a u64
```

The returned entry point becomes the RIP in the first-entry `iretq` frame, so the
capsule begins executing at its ELF entry when the scheduler first runs it
([scheduler](scheduler.md)).

The loader itself is a singleton initialised at boot
(`src/elf/loader/global.rs:26`, `init_elf_loader`). It holds an ASLR manager and
caches for loaded libraries and symbols (`src/elf/loader/core/loader/state.rs`).

---

## Parsing and validation

Loading an executable into an address space runs header parse, validation,
program-header parse, base selection, and segment load
(`src/elf/loader/core/loader/load_executable_into.rs:19`).

The header is validated before anything is mapped
(`src/elf/loader/core/parse_header/validate.rs:17`):

```
  magic is 0x7F 'E' 'L' 'F'
  64-bit class, little-endian
  current ELF version
  header sizes match the native ELF, program-header, and section-header sizes
  the section name table index is in range
  machine is EM_X86_64
  type is ET_EXEC or ET_DYN
```

A file that fails any of these is rejected rather than partially loaded. The
machine check ties the loader to the architecture; on aarch64 and riscv64 the
corresponding machine type gates instead.

## Segment rules

Each `PT_LOAD` segment is validated before it is mapped
(`src/elf/loader/core/load_segment/validate.rs:20`):

```
  p_filesz <= p_memsz                     a segment cannot claim more file than memory
  not (writable and executable)           no W and X segment
  alignment is a power of two if > 1
  the virtual address and file offset agree modulo the alignment
  the file offset plus size stays within the ELF
```

The no-write-and-execute rule is the one worth calling out. A segment cannot be
both writable and executable, so a capsule cannot ship a page it can write to and
then jump into. This is enforced at load, before the segment is mapped, so a
malformed or hostile ELF is refused rather than mapped and trusted.

## Mapping

A validated segment is mapped page by page into the target address space
(`src/elf/loader/core/load_segment/run.rs:28`):

```
  load_segment(elf, header, base, asid)
    build a load plan: alignment, page count, file offsets
    derive page permissions from the program header (execute and write bits)
    for each page: populate it in the target asid with those permissions
```

The base address is chosen by the loader's ASLR (`base_addr::load_base`), so an
`ET_DYN` capsule is placed at a randomised base rather than a fixed one, and
relative relocations are applied for the chosen base
(`src/elf/loader/core/loader/load_entry_into.rs`). The result is an `ElfImage`
recording the base, the entry point, the segments, and any dynamic, TLS, or
interpreter information present (`src/elf/loader/image/image.rs`).

---

## What the loader guarantees

By the time the loader returns an entry point, the following hold:

```
  the ELF was a well-formed 64-bit x86_64 executable or shared object
  every loadable segment fit its bounds and was not both writable and executable
  segments were mapped with exactly the permissions their headers declared
  the image was placed at a randomised base with relocations applied
```

Combined with the verified-spawn hash check that ran first, this means the code
mapped into a capsule is the exact code the manifest signed, laid out with W xor
X segment permissions, at an address the loader chose. The capsule never maps its
own code; the kernel does, from bytes it already verified.

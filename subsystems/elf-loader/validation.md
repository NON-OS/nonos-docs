# ELF Validation

Before any byte of a capsule image is mapped, its ELF header and program-header table are
validated. A malformed or unexpected image is rejected up front with a specific error, not
partway through mapping. This page documents the header checks and the program-header bounds.
The code is under `src/elf/loader/core/parse_header/`.

## Parsing the header

`parse_elf_header` (`src/elf/loader/core/parse_header/header.rs:18`) refuses an image smaller
than a header before reading anything:

```
  parse_elf_header(elf_data):
      if elf_data.len() < ElfHeader::SIZE:  FileTooSmall
      read_unaligned an ElfHeader from the front
```

The read is unaligned because the caller's buffer is arbitrary bytes; the size check ahead of
it means the read never runs off the end.

## The header checks

`validate_elf` (`parse_header/validate.rs:17`) runs ten checks in order and returns a distinct
error for each, so a rejected image says exactly why:

```
  is_valid_magic()                    else InvalidMagic          (0x7F "ELF")
  is_64bit()                          else InvalidClass          (ELFCLASS64)
  is_little_endian()                  else InvalidEndian         (ELFDATA2LSB)
  version_is_current()                else InvalidVersion
  has_native_header_size()            else InvalidHeaderSize
  has_native_program_header_size()    else InvalidProgramHeaderSize   (56 bytes)
  has_native_section_header_size()    else InvalidSectionHeaderSize   (64 bytes)
  has_valid_section_name_table_index() else InvalidIndex
  e_machine == EM_X86_64              else InvalidMachine        (value 62)
  e_type in { ET_EXEC, ET_DYN }       else InvalidType
```

The image must be a little-endian 64-bit x86_64 object, and it must be either an executable
(`ET_EXEC`) or a shared object (`ET_DYN`, the PIE form); a relocatable object or a core file is
rejected with `InvalidType`. The entry-size checks pin the program- and section-header entry
sizes to the native struct sizes, so the loader's later fixed-stride indexing into those tables
is sound by construction.

## Program-header bounds

The program-header table is where the loadable segments are described, and its extent is
bounds-checked before it is walked (`parse_header/bounds.rs:18`):

```
  program_header_bounds(elf_data, header):
      ph_offset = header.e_phoff        (checked into usize, else ProgramHeadersOutOfBounds)
      ph_count  = header.e_phnum
      if ph_count == 0:  return (nothing to load)
      if ph_entsize != sizeof(ProgramHeader):  InvalidProgramHeaderSize
      table_bytes = ph_entsize * ph_count       (checked_mul)
      table_end   = ph_offset + table_bytes      (checked_add)
      if table_end > elf_data.len():  ProgramHeadersOutOfBounds
```

Every multiplication and addition is checked, and the table end is required to lie inside the
image, so a header claiming a table that runs past the buffer is refused rather than read out of
bounds. `parse_program_header_at` (`parse_header/program_entry.rs:19`) re-checks the index
against the count and computes each entry's offset with checked arithmetic, so no individual
program-header read can escape the image either.

## The error type

Every rejection is one variant of `ElfError` (`src/elf/errors/types/state.rs:17`), a flat enum
covering the header checks above, the segment and relocation failures the [loading](segments.md)
path can raise (`SegmentDataOutOfBounds`, `MemoryAllocationFailed`, `MemoryMappingFailed`,
`WXViolation`, `AlignmentError`, `AddressOverflow`), and the dynamic-linking errors the userland
path uses. The load path converts an `ElfError` into a spawn failure; the specific variant is
logged so a rejected capsule is diagnosable.

## Source

```
  src/elf/loader/core/parse_header/header.rs         parse_elf_header, the size floor
  src/elf/loader/core/parse_header/validate.rs        the ten header checks
  src/elf/loader/core/parse_header/bounds.rs          program-header table bounds
  src/elf/loader/core/parse_header/program_entry.rs   per-entry bounds-checked read
  src/elf/errors/types/state.rs                       the ElfError enum
```

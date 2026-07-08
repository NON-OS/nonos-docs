# Load Address, ASLR, and Relocation

The loader decides where an image lands, randomizes that base for a position-independent image,
maps its segments, applies the relative relocations a PIE needs, and computes the relocated entry
point. This page documents that orchestration. The code is under
`src/elf/loader/core/loader/`.

## The load orchestration

`load_entry_into` (`src/elf/loader/core/loader/load_entry_into.rs:20`) is the whole trusted load
into a target address space, in order:

```
  load_entry_into(elf_data, target_asid):
      header    = parse_elf_header(elf_data)
      validate_elf(header)                          // the header checks
      ph_count  = program_header_bounds(...)         // bounds-checked table
      base_addr = load_base(header, aslr_manager)    // where it lands
      for each program header:
          if PT_LOAD:  load_segment(...)             // W^X gate + map + zero-fill
      apply_relative_relocations(elf_data, header, ph_count, base_addr, target_asid)
      return entry_point(header, base_addr)
```

Validation precedes any mapping, the base is chosen once, every loadable segment is mapped
through the [segment path](segments.md), and only then are relocations applied and the entry
point returned. The function takes a `target_asid`, so it maps into a specific address space, the
freshly created one for the capsule being spawned, not the current one.

## The load base and ASLR

`load_base` (`loader/base_addr.rs:19`) chooses the base from whether the image is
position-independent:

```
  load_base(header, aslr):
      if header.is_pie():  aslr.randomize_base(DEFAULT_PIE_BASE)
      else:                DEFAULT_STATIC_BASE
```

A PIE (an `ET_DYN` image, which is what capsules typically are) is loaded at a randomized base;
a non-PIE executable is loaded at its fixed base. `randomize_base` (`aslr/manager/randomize.rs:24`)
adds a random offset within the executable randomization range to the preferred base and masks it
back to a page boundary:

```
  randomize_base(preferred):
      if executable_randomization:  (preferred + random_offset(RANGE)) & !0xFFF
      else:                         preferred
```

Randomization is on by default (`aslr/manager/settings.rs`), and each load uses the loader's ASLR
manager, so a capsule's text and data land at an address an attacker cannot predict from the
image alone. The randomization is page-aligned so it does not disturb segment alignment.

## Relocations and the entry point

A PIE's absolute references are fixed up after mapping by `apply_relative_relocations`, which
walks the relocation entries and applies the relative (`R_X86_64_RELATIVE`) fixups against the
chosen base into the target address space. This is the relocation half of the
[capsule RELRO](../../security/capsules-and-trust.md) posture: the global offset table is
relocated at load, and the paging layer write-protects it afterward. Finally `entry_point`
(`loader/base_addr.rs:27`) returns the entry, relocated by the base for a PIE and taken absolute
for a non-PIE:

```
  entry_point(header, base):
      if header.is_pie():  base + header.e_entry
      else:                header.e_entry
```

The returned entry is what the spawn path installs as the capsule's initial instruction pointer.

## Source

```
  src/elf/loader/core/loader/load_entry_into.rs   the ordered load orchestration
  src/elf/loader/core/loader/base_addr.rs          load_base, entry_point
  src/elf/aslr/manager/randomize.rs                randomize_base
  src/elf/aslr/manager/settings.rs                 ASLR defaults
  src/elf/loader/core/relocate/                    the relative relocation pass
```

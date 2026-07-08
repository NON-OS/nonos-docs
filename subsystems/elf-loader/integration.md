# Spawn Integration

The ELF loader is not called directly by a capsule. It is called by the spawn pipeline, and only
after the image has been cryptographically verified. This page documents where the load sits in
the spawn sequence. The code is under `src/kernel_core/process_spawn/capsule_spawn/`.

## Verify, then load

A capsule is spawned from a VFS artifact through `load_capsule_from_vfs`
(`.../capsule_spawn/from_vfs/load.rs`), which builds a verified spec and calls `spawn_verified`
(`.../runner/verified.rs:26`):

```
  spawn_verified(spec, trust_anchor, now_ms):
      preflight::run(spec, trust_anchor, now_ms)     // manifest + certificate verification
      install(InstallParams { elf, caps, ... })
```

The ordering is the point: `preflight::run` performs the full
[verified-spawn](../../security/capsules-and-trust.md) check, the NØNOS-ID certificate against the
baked trust anchor, then the manifest against the publisher keys, with both Ed25519 and ML-DSA-65
required, before `install` runs. The ELF loader is never reached for an image that failed
verification, so mapping only ever happens on an image whose signatures and capabilities have
already been checked.

## Installing the image

`install::run` (`.../runner/install/install.rs:30`) creates the process and its address space,
then loads the ELF into it (`.../runner/install/load_elf_into_pid.rs:21`):

```
  load_elf_into_pid(elf, pid, debug_tag):
      asid = lookup_asid_for_process(pid)     else AddressSpace
      load_elf_entry_into(elf, asid)          else { log err; SpawnError::ElfLoad }
```

`load_elf_entry_into` is the loader's [entry orchestration](layout.md) targeting the new address
space's ASID. It returns the relocated entry point, which the spawn path installs as the capsule's
initial instruction pointer, and any `ElfError` becomes a `SpawnError::ElfLoad` with the specific
variant logged. Because the loader maps into `asid` rather than the current address space, the
capsule's pages are placed directly in its own fresh page tables and are never briefly visible in
the kernel's or another capsule's mapping.

## Where it sits

```
  load_capsule_from_vfs
      -> spawn_verified
          -> preflight::run        certificate + manifest verification   (must pass first)
          -> install::run
              -> create_process    fresh PCB + address space
              -> load_elf_into_pid
                  -> load_elf_entry_into(elf, asid)   validate + map + relocate
```

The ELF loader is the last trusted step that turns verified bytes into an executable address
space. The verification that gates it is documented on the
[capsules and trust](../../security/capsules-and-trust.md) page, the address space it maps into is
covered on the [process](../process/pcb.md) and [memory](../memory/paging-manager.md) pages, and
the capabilities the spawn installs are the [capability model](../../security/capabilities-and-tokens.md).

## Source

```
  src/kernel_core/process_spawn/capsule_spawn/from_vfs/load.rs           load_capsule_from_vfs
  src/kernel_core/process_spawn/capsule_spawn/runner/verified.rs          spawn_verified (verify-then-install)
  src/kernel_core/process_spawn/capsule_spawn/runner/install/install.rs   process + address space + load
  src/kernel_core/process_spawn/capsule_spawn/runner/install/load_elf_into_pid.rs
```

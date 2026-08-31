# Userspace Init

Once the core is ready, the kernel creates one process, `init`, enters user space, and from there the
whole system is capsules. Init is the supervisor: it spawns the capsule fleet in dependency order and
then stays resident to watch over them. This page documents that transition. The code is
`src/kernel_core/init/entry/microkernel_main.rs` and `src/userspace/`.

## Creating init

`microkernel_main` (`src/kernel_core/init/entry/microkernel_main.rs:22`) is where the kernel crosses into
user space:

```
  microkernel_main():                                    microkernel_main.rs
      spin-wait a short settling window (~2.5 s guard)    :28
      init_pid = create_process("init", Running, High)    :38
      create_address_space(init_pid)                      :48
      allocate_kernel_stack(init_pid)                     :53
      CURRENT_PID = init_pid                              :58
      userspace::run_init()                               :61
```

Init is created like any process, a [PCB](../process/pcb.md), a fresh [address space](../memory/unified-vm.md),
and a kernel stack, at high priority because it is the supervisor. Each of these steps is fatal on
failure: a system that cannot create its init process cannot boot, so the failure halts with a reason
rather than continuing. Once init is current, control enters `run_init`, which is the first code that
runs conceptually in user space.

## Spawning the fleet

`run_init` (`src/userspace/init/entry.rs:20`) spawns the capsule fleet in dependency order:

```
  run_init():                              src/userspace/init/entry.rs
      spawn_ramfs()               :26      the filesystem first (the first capsule)
      spawn_core_after_ramfs()    :27
      spawn_display_core()        :28
      spawn_drivers()             :29      bus, input, NIC drivers
      spawn_vfs()                 :30
      spawn_network()             :31      net_core, sockets, nym
      spawn_desktop()             :32      compositor, window manager, shell
      spawn_market()              :33
      spawn_apps()                :34
      lower_init_priority()       :39
      init_loop()                 :42      the supervisor loop
```

The per-group plans live under `src/userspace/init/spawn_plan/`, and each plan entry funnels into a
`capsule_boot::boot()` call (`src/userspace/init/capsule_boot/`), which spawns the capsule and registers
it on success. The initial set is the statically compiled, feature-gated spawn plan, not an external
manifest read at boot.

The order is a dependency order: the [filesystem](../storage/README.md) comes up first because later
capsules read from it, the [drivers](../hardware-broker/README.md) before the stacks that use them, the
[network](../networking/README.md) and [display](../graphics/README.md) before the desktop, and the
applications last. Each spawn goes through the [verified-spawn](../elf-loader/integration.md) pipeline,
so every capsule in the fleet is signature- and capability-checked as it is loaded. After the fleet is
up, init lowers its own priority so it does not compete with the capsules it launched.

## The supervisor loop

Init does not exit; it becomes the supervisor (`src/userspace/init/supervisor/`). The `init_loop` is
the resident parent that the [process lifecycle](../process/lifecycle.md) reaps children into: when a
capsule exits, init is where the final teardown accounting settles, and a supervised capsule that dies
can be restarted per policy. This is the userspace analogue of the kernel's fatal invariants, the
system always has a live supervisor, just as it always has a keyed IPC path and a working VM. From here
the system is running: the kernel holds the primitives, and the capsule fleet, filesystem, network,
display, and applications, does the work.

## Security analysis

Userspace init is where the trust chain reaches the capsules, so its properties are about what a capsule
must prove before it is allowed to run and what happens to the system if the proof fails.

**The spawn is fail-closed on attestation.** Every enrolled capsule goes through the verified-spawn
pipeline, and the STARK attestation gate (`attest_gate`, `src/kernel_core/process_spawn/capsule_spawn/runner/attest_gate.rs:23`)
is the last check before install. It reads the capsule's `NZKCAPS1` attestation trailer
(`capsule_spawn/spec.rs:39`). By default the gate is closed: a capsule with an empty trailer is rejected
with `SpawnError::AttestationRejected` (`attest_gate.rs:34`), and otherwise the trailer is verified by
`verify_capsule_attestation` (`attest_gate.rs:38`, impl `src/security/capsule_attest/verify.rs:33`) bound
to the ELF and the caps being installed. Only the `nonos-zk-rollout` build feature turns the gate
permissive, logging the failure but returning `Ok`; a production build without that feature will not
spawn an unattested or wrongly attested capsule. The proved measurement is then recorded against the new
pid (`record_attested`, `runner/verified.rs:66`), never recomputed. Because init spawns the whole fleet
through this one pipeline, a capsule that cannot prove its attestation simply does not join the system.

**Attestation is one layer of a stacked check, not the only one.** The spawn also verifies the NONOS id
certificate against the baked trust anchor, the manifest against the publisher signature and the
capability ceiling, and the payload hash, before the attestation gate runs (the `SpawnError` variants in
`capsule_spawn/spec.rs` name each: `NonosIdCertRejected`, `ManifestRejected`, `AttestationRejected`).
The attestation gate is the innermost of these, so a capsule reaching it has already passed cert and
manifest verification.

**The system always has a live supervisor.** Init does not exit; it lowers its own priority and enters
`init_loop` (`src/userspace/init/supervisor/`). This is the userspace analogue of the kernel's fatal
invariants: just as the kernel refuses to run without a keyed IPC path or a working VM, the running
system always has a resident parent that reaps and can restart supervised capsules. The honest boundary
is that restart is per policy, not unconditional, so a capsule the supervisor is not configured to
restart stays down after it exits.

## Debugging userspace init

The fleet spawn is a serial-log stream, since the on-screen kernel log stays off (see
[kernel init](kernel-init.md)). Init's own progress prints through `boot_log::ok` as "[UKERNEL]
Creating init" and "[UKERNEL] Entering userspace" (`src/kernel_core/init/entry/microkernel_main.rs`), so
those two lines confirm the kernel crossed into user space at all. Each capsule attestation prints one
line from the gate (`attest_gate.rs`): "[ZK-ATTEST] ok <name>" on success, "[ZK-ATTEST] FAIL <name>:
<reason>" when verification fails, and "[ZK-ATTEST] none <name>" when a capsule has no trailer. A spawn
that is refused prints
"[RUNTIME-LOAD] FAILED name=<name> reason=<reason>" from the load path
(`capsule_spawn/from_vfs/load.rs`), where the reason string names the exact stage: "attestation" for the
ZK gate, "manifest:pub_sig" or the other "manifest:*" reasons for manifest checks, and "elf_load" or
"address_space" for the mechanical steps. The classic failure mode is a system that boots to the
bootloader splash and then stalls with no desktop: in a fail-closed build this is usually the
attestation gate refusing the fleet, which is what "[ZK-ATTEST] FAIL" or "[ZK-ATTEST] none" on the
first capsules confirms. Enrollment is a build-time concern (the `NONOS_DEV=1` make targets mint the
dev attestation material); a build whose capsules were never enrolled will show the "none" line and,
without `nonos-zk-rollout`, refuse every spawn.

## Source map

```
  src/kernel_core/init/entry/microkernel_main.rs                   microkernel_main, create init, enter userspace
  src/userspace/init/entry.rs                                       run_init, the fleet spawn order
  src/userspace/init/spawn_plan/                                    the per-group spawn plans
  src/userspace/init/capsule_boot/                                  the per-capsule spawn + service register
  src/userspace/init/supervisor/                                    the resident supervisor loop
  src/kernel_core/process_spawn/capsule_spawn/runner/attest_gate.rs the ZK attestation gate and its markers
  src/kernel_core/process_spawn/capsule_spawn/spec.rs               CapsuleSpecVerified and the SpawnError variants
  src/kernel_core/process_spawn/capsule_spawn/from_vfs/load.rs      the RUNTIME-LOAD failure marker and reason strings
```

Every reference above is verified against those trees. The core init that runs before this transition is
on the [kernel init](kernel-init.md) page; the full capability and manifest verification the spawn layers
around the attestation gate is on the [verified-spawn](../elf-loader/integration.md) and
[capsule trust](../../security/capsules-and-trust.md) pages.

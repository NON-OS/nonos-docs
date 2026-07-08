# Unified Virtual Memory Init

`init_unified_vm` is the single step that takes the kernel from running on the
page tables the bootloader handed it to running entirely from its own upper half,
with the bootloader's low-half identity map torn down. It runs once, before any
address space is created, and every one of its steps is a precondition for
creating one. If any step fails, it returns a specific error string and the kernel
fails loudly with a deterministic reason rather than letting a later
process-creation site surface a swallowed fault. The code is
`src/memory/unified/init/run.rs`.

## The upper and lower half

The virtual address space is split in two. The kernel half is PML4 entries 256
through 511 (`KERNEL_HALF_START..KERNEL_HALF_END`, `run.rs:27`), holding the direct
map, the kernel text, and the kernel heap; it is shared and mapped identically into
every address space. The lower half, PML4 entries 0 through 255, is where the
bootloader placed an identity map to get the kernel into Rust, and later where each
capsule's private user memory lives. This function's job is to confirm the kernel
half is properly populated and then remove the bootloader's lower-half identity, so
that afterward a stray lower-half pointer dereferenced in kernel mode faults instead
of silently reading boot leftovers. That is what "RAM-resident" means in practice:
there is no lower-half scratch the kernel falls back on.

## The sequence

The function is idempotent: it swaps a `VM_UNIFIED_INITIALIZED` flag and returns
early if it has already run (`run.rs:36`). Otherwise it runs six steps in a fixed
order, each a real precondition for the next.

```
  1  paging manager init        read CR3, register the kernel address space
  2  confirm active CR3         a page-table root must be recorded
  3  confirm kernel AS          the address-space map must be non-empty
  4  confirm kernel half        PML4[256..512] must hold bootloader mappings
  5  probe the frame allocator  allocate and free one frame to prove it works
  5b page allocator init        bring up kernel virtual-range allocation
  5c LAPIC rebind               remap the LAPIC into the kernel half
  6  tear down the low half     drop PML4[0..256] once the kernel half is proven
```

Step 1 initialises the paging manager, which reads the current CR3 and registers
the kernel address space. Steps 2 and 3 confirm that succeeded: there must be an
active page-table root recorded and at least one address space registered, or the
function returns `"no active page table after manager init"` or
`"kernel address space not registered"`.

Step 4 is the guard the function exists for. `create_address_space` clones PML4
entries 256 through 511 from the active table into each new address space, so if
the kernel half of the active table is empty, that clone produces a table with no
kernel mappings and later faults. The step counts the populated kernel-half entries
(`count_pml4_entries` over `256..512`), logs the count, and returns
`"bootloader CR3 has no kernel-half PML4 entries"` if the count is zero. This is
explicitly the regression it is protecting against.

Step 5 proves the [frame allocator](physical-frames.md) can hand out a page by
allocating one and freeing it, and step 5b brings up the higher-level
[page allocator](page-allocator.md) that carves kernel virtual ranges for things
like per-process kernel stacks, which is only safe now that the frame allocator is
up and `map_page` is live.

## The LAPIC rebind

Step 5c (`run.rs:87`) is an ordering hazard the code calls out. The Local APIC is
addressed by `sys::apic` at its raw physical base `0xFEE00000`, and that address is
only reachable while the bootloader's low identity map is still live. Step 6 is
about to tear that map down, after which any `eoi()` or timer access to the LAPIC
would page fault at `cr2=0xFEE000B0`. So before the teardown, this step maps the
LAPIC page into the kernel half as an uncached device mapping
(`mmio::map_device_memory`) and atomically republishes the LAPIC base to that new
virtual address (`apic::rebind_to_virt`). The rebind must happen before the
teardown, not after, which is why it is step 5c and not a later cleanup.

## The low-half teardown

Step 6 (`run.rs:99`) removes the bootloader's low-half identity, but only when it
is safe. If the kernel half has at least two populated entries, meaning both the
direct map at PML4[256] and the kernel text at PML4[511] are present, the kernel can
run entirely from the upper half and `clear_low_half` zeroes PML4[0..256]. If only
one entry is populated, the kernel is in the legacy low-half layout where PML4[0] is
still its own text, and the teardown is skipped so the kernel does not unmap the
code it is executing. After a successful teardown the kernel executes purely from
the upper half and the identity map is gone.

## Failure and determinism

Every step that can fail returns a distinct `&'static str`, and the whole function
returns `Result<(), &'static str>`, so a failure names exactly which precondition
was not met. The comment at the top states the intent: fail loudly with a
deterministic reason rather than letting a swallowed error surface later at a
process-creation site. Combined with the idempotence guard, the function either
brings the unified VM fully up or reports precisely why it could not.

## Where this connects

After this runs, the [paging manager](paging-manager.md) can create address spaces
by cloning the kernel half, the [heap](heap.md) and page allocator are live, and
the LAPIC is reachable through its permanent kernel-half mapping. This function is
called from `microkernel_init` in the [boot sequence](../boot/README.md), after core init and
before process management is brought up.

## Source

```
  src/memory/unified/init/run.rs             init_unified_vm, the ordered sequence
  src/memory/unified/init/clear_low_half.rs  the low-half teardown
  src/memory/unified/init/count_pml4_entries.rs  the kernel-half check
```

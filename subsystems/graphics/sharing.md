# Sharing and Attaching

A surface becomes useful when more than one capsule can see it: a client draws into it and a
compositor reads it to build the screen. NØNOS shares surfaces by mapping the same physical frames
into a second capsule's address space, so the sharing is zero-copy, and it tracks the sharing with
a refcount and a per-receiver attach map. This page documents share, attach, and release. The code
is under `src/kernel_core/surface_registry/share/` and `.../release/`.

## Share

`share_surface` (`share/share_surface.rs:20`) is the owner marking a surface available and bumping
its refcount:

```
  share_surface(owner_pid, handle):
      decode handle; verify epoch (BadHandle) and owner (NotOwner)
      refcount += 1 (checked)
      return handle
```

Only the owner can share, and the epoch is checked, so a capsule cannot share a surface it does not
own or a stale handle. The refcount is what keeps the frames alive while any sharer holds them.

## Attach

`attach_surface` (`share/attach_surface.rs:25`) is the receiving side: it maps the surface's frames
into the receiver's address space and hands back the descriptor and the virtual address:

```
  attach_surface(receiver_pid, handle, out_desc):
      if already attached (attach_map lookup):  return the recorded VA   // idempotent
      if receiver is the owner and has a base VA:  return owner_base_va   // self-attach, no remap
      else:
          refcount += 1; clone the frame list
          reserve a VMA in the receiver of frames.len() pages
          map each frame into the receiver's ASID with user read-write perms
          record (receiver, handle) -> (VA, len) in the attach map
          return the VA
```

The general case maps the surface's physical frames into a freshly reserved region of the
receiver's address space, one page per frame, so both capsules now have the same physical pixels
mapped: a write by the client is visible to the compositor with no copy. Two special cases avoid
redundant work: a repeat attach returns the VA already recorded for that receiver and handle
(idempotent), and a self-attach by the owner returns the VA the owner already registered the
surface at rather than creating a second mapping, which would leave a virtual address with no
backing VMA and break present. The mapping uses the [paging manager](../memory/paging-manager.md)
per-ASID map, and the receiver's VA comes from its own VMA reservation, so the kernel never guesses
a user address.

## Release

Surfaces are released two ways (`release/`): `release_surface` drops a reference (and frees the
slot and frames when the last reference goes), and `release_owned_by_pid` is called on process exit
to drop every surface a capsule owned. The attach map is likewise cleaned per handle and per pid
(`attach_map/forget*`), so a capsule that exits does not leave a surface slot or a stale attachment
behind. Combined with the epoch on the handle, this means a surface's lifetime is bounded by its
references and its owner, and a handle to a released surface fails rather than aliasing a new one.

## Source

```
  src/kernel_core/surface_registry/share/share_surface.rs    share (owner, refcount)
  src/kernel_core/surface_registry/share/attach_surface.rs   attach (cross-ASID frame mapping)
  src/kernel_core/surface_registry/attach_map/               per-receiver VA record and cleanup
  src/kernel_core/surface_registry/release/                  release_surface, release_owned_by_pid
```

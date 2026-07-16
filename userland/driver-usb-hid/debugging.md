# Debugging capsule_driver_usb_hid

This page lists the markers the driver and its input path emit, and the concrete failure modes with
where to look for each. For the driver model see the [README](README.md), the
[service protocol](protocol.md), the [enumeration](enumeration.md), and the
[input-post path](input-post.md) pages in this folder.

## Markers

The first live marker is enumeration. Once at least one HID endpoint binds, the poll loop emits
`[USB-HID-ENUM] tablet bound` through `mk_debug`, exactly once (`src/orchestrator/poll/run.rs:35`). If
that line never appears, no HID device was enumerated: the capsule is either still blocked waiting for
`driver.xhci0` to appear (`src/orchestrator/run.rs:20`), or enumeration found no connected port with a
boot HID interface (`src/orchestrator/enumerate/run.rs:31`). Because this capsule holds no device
authority, an absent marker points upstream at xHCI discovery or the port state, not at HID parsing.

Two kernel-side markers confirm the post path. The kernel emits the one-shot bench marker
`input_post_first` on the very first successful post into the ring
(`src/kernel_core/surface_registry/input_ring.rs:68`) and `input_drain_first` on the first drain by
the router (`src/syscall/dispatch/router/input_ops.rs:79`). `input_post_first` present but the desktop
still dead means events are entering the ring but the router or focus path is the suspect, covered in
the [input path debugging](../../subsystems/input/path.md); `input_post_first` absent after
`[USB-HID-ENUM] tablet bound` means the parser is running but every post is failing, which shows up as
a rising post-failure count in `OP_GET_STATE` (`src/server/handlers/get_state.rs:28`).

## Failure modes

### Device not enumerated

No `[USB-HID-ENUM] tablet bound`. First confirm `driver.xhci0` is live: the lookup loop yields forever
until the service resolves, so the driver simply never leaves that loop if the transport is absent
(`src/xhci/lookup.rs:21`, `src/orchestrator/run.rs:20`). If the transport is up, the device must
present a boot-subclass HID interface on an interrupt IN endpoint, which is the only shape
`HidBinding::from_pair` accepts; a HID interface that is not on an interrupt IN endpoint, or a
non-HID-class interface, yields no binding (`src/descriptors/binding.rs:32`,
`src/descriptors/binding.rs:33`). The `configure_port` path also bails silently if enable-slot,
address-device, the descriptor read, or the sanity check on slot/port/speed fails
(`src/orchestrator/enumerate/configure_port.rs:29`, `src/orchestrator/enumerate/configure_port.rs:35`),
so a device that enumerates on another OS but not here points at one of those transport ops. Probe a
captured descriptor offline with `OP_PROBE_CONFIG`: `E_NO_HID` (-61) means the descriptor is valid but
carries no boot HID interface (`src/server/handlers/probe_config.rs:37`).

### No input despite enumeration

The endpoint is bound but reports are not arriving or not posting. `interrupt_in` returning `E_AGAIN`
(-11) continuously means the transport has no completed report to hand over, which is an xHCI-side
issue, not a HID one (`src/xhci/ops/interrupt_in.rs:38`). If reports are arriving but nothing reaches
the desktop, read `OP_GET_STATE`: a rising key or mouse report count with a rising post-failure count
means the parser runs but the post fails, so the ring is full or the `InputSource` gate is denying the
post (`src/server/handlers/get_state.rs:23`, `src/syscall/contract/cap_table/mk.rs:78`). A rising
report count with zero post failures and a dead desktop moves the suspect downstream to the router or
focus, on the [input path](../../subsystems/input/path.md) page.

### Wrong keycodes or characters

The usage-to-ASCII mapping is `src/hid/keymap.rs:26` and the punctuation table is
`src/hid/punctuation.rs:17`; the navigation-key and flag mapping is `src/hid/post_key.rs:37`. A shifted
or Caps Lock mismatch is the `shift XOR caps` decision for letters (`src/hid/keymap.rs:29`). A key that
produces no ASCII is posted as `0x2000 | scancode` (`src/hid/post_key.rs:54`), so a consumer seeing
those high codes is receiving a key the keymap does not cover, not a corrupt event. A stuck or
repeating key would show in the diff: `Keyboard::feed` remembers the previous frame and only emits on a
change, so a repeat points at the report stream, not the parser (`src/hid/keyboard/feed.rs:25`).

### Malformed descriptor rejected

`OP_PROBE_CONFIG` returns `E_INVAL` (-22) for a body over 512 bytes or a descriptor whose header or
record lengths are inconsistent (`src/server/handlers/probe_config.rs:25`,
`src/descriptors/parse.rs:32`), and `E_NO_HID` (-61) for a valid descriptor with no boot HID interface
(`src/server/handlers/probe_config.rs:37`). The same walk gates live enumeration, so a descriptor that
probes as `E_INVAL` will also fail to bind on real hardware.

## Source map

```
  src/orchestrator/poll/run.rs                     [USB-HID-ENUM] tablet bound, once on first bind
  src/orchestrator/run.rs                          the yield loop waiting for driver.xhci0
  src/xhci/lookup.rs                               resolve driver.xhci0 by name
  src/xhci/ops/interrupt_in.rs                     E_AGAIN means no report pending
  src/descriptors/binding.rs                       the only accepted device shape
  src/descriptors/parse.rs                         the descriptor validation
  src/hid/keyboard/feed.rs                         the change-only keyboard diff
  src/hid/keymap.rs                                usage -> ASCII, shift/caps XOR
  src/hid/punctuation.rs                           the punctuation table
  src/hid/post_key.rs                              navigation codes and the 0x2000 fallback
  src/server/handlers/get_state.rs                 the counters and post-failure fields
  src/server/handlers/probe_config.rs              E_INVAL / E_NO_HID for a probed descriptor
  src/syscall/contract/cap_table/mk.rs             the InputSource gate on the post
  src/kernel_core/surface_registry/input_ring.rs   input_post_first
  src/syscall/dispatch/router/input_ops.rs         input_drain_first
```

Every reference above is verified against those trees.

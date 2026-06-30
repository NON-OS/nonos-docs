# Driver Broker ABI

The driver broker ABI is the kernel boundary used by driver capsules to claim a
device and request the narrow hardware grants needed to drive it. It is not a
general physical-memory API and it is not a kernel driver framework.

## Authority model

Every broker operation is capability checked and device scoped. A driver must
first claim a device. The claim records the owner process and an epoch. Later
grant calls must present the current claim and epoch, which prevents replaying a
grant against a device that has been released and claimed by another capsule.

## Capabilities

| Capability | Authority |
|---|---|
| `DeviceEnum` | enumerate visible devices |
| `Driver` | claim and release a device |
| `Mmio` | map and unmap vetted device BAR ranges |
| `Irq` | bind, poll, acknowledge, and release interrupt grants |
| `Dma` | pin driver buffers and expose broker-vetted DMA addresses |
| `Pio` | x86_64-only port IO grants and port access |

## Claim lifecycle

```text
MkDeviceList(out)
MkDeviceClaim(device) -> claim_id, epoch
MkDeviceRelease(device, claim_id, epoch)
```

Release invalidates the ownership epoch and drops grants owned by the claim.
Grant handlers reject calls from processes that do not own the claim or present a
stale epoch.

## MMIO grants

```text
MkMmioMap(device, claim_id, epoch, bar, offset, length, out: MmioMapOut)
MkMmioUnmap(grant)
```

The broker validates the requested region against the claimed PCI device record
before mapping it into the caller. A driver can map only BAR space belonging to
the device it owns.

## IRQ grants

```text
MkIrqBind(device, claim_id, epoch, gsi, flags, vector_count, out: IrqBindOut)
MkIrqPoll(grant, out: IrqPollOut)
MkIrqAck(grant)
MkIrqUnbind(grant)
```

INTx grants route a GSI to a broker vector and keep the line masked between
delivery and acknowledgement. MSI-X grants allocate a contiguous vector/grant
range for the requested vector count.

## DMA grants

```text
MkDmaMap(device, claim_id, epoch, user_buffer, length, flags, out: DmaMapOut)
MkDmaUnmap(grant)
```

The broker pins the caller buffer, bounds the mapping by device class, and
returns the DMA address. Unmap releases the pin and invalidates the grant.

## PIO grants

```text
MkPioGrant(device, claim_id, epoch, base, length, out)
MkPioRead(grant, port, width, out)
MkPioWrite(grant, port, width, value)
MkPioRelease(grant)
```

PIO exists only on x86_64. Reads and writes are rejected unless the target port
falls inside the caller's grant and uses a supported width.

## Security invariants

- No broker call grants access without both capability and current device
  ownership.
- Grants are scoped to one owner process and one device claim epoch.
- MMIO grants are constrained to the claimed device BARs.
- IRQ grants are owned, polled, acknowledged, and released by grant id.
- DMA grants pin bounded caller memory and are explicitly unmapped.
- PIO grants are x86_64-only and range checked on every access.

## Non-goals

The broker ABI does not parse device protocols, implement NIC/storage/GPU
drivers in the kernel, persist hardware state, or allow arbitrary physical
memory access. Protocol state belongs in driver capsules and higher service
capsules.

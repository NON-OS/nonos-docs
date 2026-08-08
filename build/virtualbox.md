# Running NØNOS 0.9.1 in VirtualBox

The VirtualBox defaults will not boot this image. It is UEFI only and VirtualBox
creates machines with a legacy BIOS. Below is a configuration that is known to
work, and the commands to create it exactly.

Verified: VirtualBox 7.1.6 on macOS, full desktop at 1920x1080, no rendering
faults, on both graphics controllers. Windows is not verified; the section at the
end covers what differs there.

## The settings that matter

| Setting | Value | Why |
|---|---|---|
| EFI | **enabled** | UEFI only, there is no legacy boot path |
| Memory | 4096 MB | the capsule set and desktop want room |
| CPUs | 2 | |
| Graphics controller | VMSVGA | VBoxSVGA also works |
| Video memory | 128 MB | 16 MB also worked, 128 is comfortable |
| Network adapter | Intel PRO/1000 MT Desktop (82540EM) | this is the card the driver drives |
| 3D acceleration | **off** | |

## Create it from the command line

### Windows

`VBoxManage.exe` lives in `C:\Program Files\Oracle\VirtualBox`. From PowerShell:

    cd "C:\Program Files\Oracle\VirtualBox"

    .\VBoxManage.exe createvm --name nonos --ostype Linux_64 --register
    .\VBoxManage.exe modifyvm nonos --memory 4096 --cpus 2 --firmware efi `
        --graphicscontroller vmsvga --vram 128 --nic1 nat --nictype1 82540EM `
        --accelerate3d off
    .\VBoxManage.exe storagectl nonos --name SATA --add sata --controller IntelAhci
    .\VBoxManage.exe storageattach nonos --storagectl SATA --port 0 --device 0 `
        --type dvddrive --medium "C:\path\to\nonos-0.9.1.iso"
    .\VBoxManage.exe startvm nonos

### macOS and Linux

    VBoxManage createvm --name nonos --ostype Linux_64 --register
    VBoxManage modifyvm nonos --memory 4096 --cpus 2 --firmware efi \
        --graphicscontroller vmsvga --vram 128 --nic1 nat --nictype1 82540EM \
        --accelerate3d off
    VBoxManage storagectl nonos --name SATA --add sata --controller IntelAhci
    VBoxManage storageattach nonos --storagectl SATA --port 0 --device 0 \
        --type dvddrive --medium /path/to/nonos-0.9.1.iso
    VBoxManage startvm nonos

## Windows: Hyper-V is worth checking, but it is not proven to be the cause

Treat this as a thing to rule out rather than the answer. Our own verified-good
machine on macOS runs through the same kind of fallback, using Apple's hypervisor
framework instead of its own driver, and renders perfectly. So a fallback by
itself does not produce the reported symptoms. What differs on Windows is that
the fallback goes through Hyper-V, which is a different implementation, and that
is untested by us.

It remains a common cause of a VirtualBox machine that boots but responds slowly
or appears frozen, and it is not specific to NØNOS.

If Hyper-V, WSL2, Virtual Machine Platform, Windows Sandbox, or Core Isolation
memory integrity is enabled, VirtualBox cannot use its own hypervisor. It falls
back to running through Microsoft's interface, and that path is slow and
unreliable. A **turtle icon** appears in the machine window's status bar when
this is happening.

Check whether a hypervisor is already running:

    systeminfo | findstr /i "hyper-v"

"A hypervisor has been detected" means VirtualBox is in the fallback.

Confirm from the machine's own log, Machine menu then Show Log, or:

    Select-String -Path "$env:USERPROFILE\VirtualBox VMs\nonos\Logs\VBox.log" -Pattern "NEM|VT-x|AMD-V"

`NEM` means the fallback. `VT-x` or `AMD-V` means VirtualBox has the processor to
itself, which is what you want.

To turn it off, in an **administrator** PowerShell, then reboot:

    bcdedit /set hypervisorlaunchtype off
    Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -NoRestart
    Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -NoRestart
    Disable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -NoRestart
    Disable-WindowsOptionalFeature -Online -FeatureName Containers-DisposableClientVM -NoRestart

Then turn off Windows Security, Device security, Core isolation, Memory
integrity, and reboot again. Every one of these has to be off; one is enough to
force the fallback.

This disables WSL2 and Docker Desktop's default backend. To put it back:

    bcdedit /set hypervisorlaunchtype auto

## If it still misbehaves

Send us three things and we can usually name the cause without guessing:

1. `VBox.log` from the run that failed. It records the execution engine, the
   firmware mode, and every device as it initialises.
2. The SHA-256 of the ISO, so we know which build.
3. What you see: no boot at all, boot then freeze, desktop but no network, or
   graphical corruption. They have different causes.

## Known, in this release

Networking inside VirtualBox is not working in 0.9.1 and the fix is in progress.
The card is found and claimed, and the network stack does not come up behind it.
This is ours, not a VirtualBox setting, and it is unrelated to the display
symptoms above.

# Windows 7 + RTX 3050 Mobile passthrough on Fedora 43 (Acer Nitro)

A working KVM/libvirt configuration for running Windows 7 Ultimate x64 with full GPU passthrough of an NVIDIA RTX 3050 6GB Mobile on an Acer Nitro ANV15-41 laptop. Aero, Direct3D, and gaming work properly inside the VM.

This was painful to figure out — Win7 + Ampere + laptop NVIDIA + locked Acer firmware is a stack with multiple compatibility walls. This README walks through the working config and the gotchas.

## Why would anyone do this?

- Game streaming host (Apollo/Sunshine + Moonlight) running an OS that's lighter than Win10/11
- Running Win7-only software with full GPU acceleration
- Because you can

If you just want a streaming host, **Windows 10 IoT Enterprise LTSC 2021** is genuinely easier. Use that unless you specifically need Win7.

## Hardware tested

- **Laptop:** Acer Nitro ANV15-41
- **CPU:** AMD Ryzen 5 6600H (Zen 3+, 6 cores / 12 threads)
- **Discrete GPU (passed through):** NVIDIA RTX 3050 6GB Mobile (Ampere, GA107BM, PCI ID `10de:25ac`)
- **iGPU (host display):** AMD Radeon 660M
- **Host OS:** Fedora Linux 43, kernel 6.19, KDE Plasma 6
- **Guest OS:** Windows 7 Ultimate x64 SP1 + UpdatePack7R2

## Prerequisites on the host

### 1. Enable IOMMU + VFIO

Add to your kernel cmdline (edit `/boot/loader/entries/<your-entry>.conf`, append to `options` line):
amd_iommu=on iommu=pt vfio-pci.ids=10de:25ac,10de:2291 rd.driver.pre=vfio-pci rd.driver.blacklist=nouveau,nvidia,nvidia_drm,nvidia_modeset,nvidia_uvm modprobe.blacklist=nouveau,nvidia,nvidia_drm,nvidia_modeset,nvidia_uvm pci=realloc=on video=efifb:off

Replace `10de:25ac,10de:2291` with your GPU + audio function PCI IDs from `lspci -nn`.

Reboot. Verify:

```bash
lspci -nnk -s 01:00
```

Both functions should show `Kernel driver in use: vfio-pci`.

### 2. Required packages

```bash
sudo dnf install -y libvirt qemu-kvm virt-install virt-manager edk2-ovmf
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt $USER
# log out and back in
```

## Installing Windows 7

### 1. Get the Win7 SP1 x64 ISO

The vanilla Microsoft Win7 Ultimate SP1 x64 ISO works. Avoid heavily-modified ISOs like SiMPLiXED ESU — they triggered `0x000000F7 DRIVER_OVERRAN_STACK_BUFFER` BSODs during install in our testing.

### 2. Get the virtio-win drivers

```bash
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.285-1/virtio-win-0.1.285.iso
```

Win7 is end-of-life for virtio-win — version 0.1.285 is the last that fully supports it.

### 3. Create the VM (no GPU passthrough yet)

```bash
sudo virt-install \
  --name win7 \
  --memory 16384 \
  --vcpus 8 \
  --cpu host-passthrough,check=none \
  --machine q35 \
  --boot hd,cdrom \
  --disk path=/var/lib/libvirt/images/win7.qcow2,size=128,format=qcow2,bus=virtio,cache=none,io=native,discard=unmap \
  --disk path=/path/to/Win7_SP1_x64.iso,device=cdrom,bus=sata \
  --disk path=/path/to/virtio-win-0.1.285.iso,device=cdrom,bus=sata \
  --network network=default,model=e1000e \
  --graphics spice \
  --video qxl \
  --controller type=usb,model=nec-xhci \
  --memballoon none \
  --features hyperv.relaxed.state=on,hyperv.vapic.state=on,hyperv.spinlocks.state=on,hyperv.spinlocks.retries=8191 \
  --clock hypervclock_present=yes \
  --os-variant win7 \
  --check disk_size=off \
  --noautoconsole
```

### 4. Install Windows

- At disk selection: **Load driver** → browse to virtio-win CD → `viostor\w7\amd64`
- Continue installation normally
- After Windows is at the desktop, install remaining virtio drivers from Device Manager (yellow warning icons) — Network (NetKVM), serial (vioserial). **Don't try `virtio-win-guest-tools.exe`** — needs Win10+.

### 5. Update Windows with UpdatePack7R2

Download UpdatePack7R2 directly inside the VM:
https://update7.simplix.info/UpdatePack7R2.exe

Run with:
UpdatePack7R2.exe /ie11 /reboot

Walk away for 30+ minutes. Multiple reboots happen automatically. This brings you to the latest ESU-patched Win7 state.

### 6. Shut down Windows cleanly

## Adding GPU passthrough

Once Win7 is fully installed and updated, shut down the VM and edit the XML:

```bash
sudo virsh edit win7
```

Apply these changes:

### Add the qemu namespace to the root tag

```xml
<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">
```

### Add anti-VM detection inside `<features>`

Inside the existing `<hyperv>` block:
```xml
<vendor_id state="on" value="randomid"/>
```

Anywhere in `<features>`:
```xml
<kvm>
  <hidden state="on"/>
</kvm>
```

### Bump xhci ports

```xml
<controller type="usb" index="0" model="nec-xhci" ports="15">
```

### Add the GPU + audio hostdevs

This is the critical fix. **Win7 q35 has bridge enumeration bugs that cause Code 12 when GPUs are placed behind pcie-root-ports.** Place them directly on pcie.0 (bus `0x00`) at slot `0x10` as multifunction:

```xml
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x10" function="0x0" multifunction="on"/>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x1"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x10" function="0x1"/>
</hostdev>
```

Adjust the source addresses to match your GPU's actual location from `lspci`.

### Add MMIO window expansion at the bottom

Just before `</domain>`:

```xml
<qemu:commandline>
  <qemu:arg value="-global"/>
  <qemu:arg value="q35-pcihost.pci-hole64-size=512G"/>
</qemu:commandline>
```

Required for ReBAR-enabled Ampere GPUs.

Save and start the VM.

## Installing the NVIDIA driver in Win7

This is the most fragile part. Win7's last official NVIDIA driver is **475.14** (Ampere/30-series support requires modding).

### INF-only mod

For some Ampere SKUs, an INF mod is enough — add the device ID line to the appropriate section in `nv_dispi.inf`. For our RTX 3050 6GB Mobile (`25AC`):
%NVIDIA_DEV.25AC% = Section165, PCI\VEN_10DE&DEV_25AC

In the `[Strings]` section:
NVIDIA_DEV.25AC = "NVIDIA GeForce RTX 3050"

### Driver signing

Win7 enforces driver signatures. For modded drivers, enable test signing:

```cmd
bcdedit /set testsigning on
```

Reboot. "Test Mode" watermark appears — that's expected.

### Install the driver

Run the modded installer. After install + reboot, check Device Manager:
- **NVIDIA GeForce RTX 3050** with status "This device is working properly" → Done
- **Code 43** → INF-only mod isn't enough for your specific GPU silicon; you may need binary patches in `nvlddmkm.sys`, which is much harder. Look into the [Win-Raid forum](https://winraid.level1techs.com/) for community-built drivers.

## Verifying it works

- Aero theme should be enabled (DWM uses Direct3D)
- `dxdiag` should show NVIDIA RTX 3050 under Display tab
- SPICE viewer goes black when NVIDIA takes over as primary display
- Physical monitor connected to the laptop's HDMI/DP output should light up showing Windows

## Troubleshooting

### Code 12 ("Not enough resources")

→ GPU not on `pcie.0` slot 0x10 multifunction. Apply that placement.

### Code 43 ("Device cannot start")

Multiple causes:
1. NVIDIA detected VM. Add `<vendor_id>` and `<kvm hidden>`.
2. Modded driver doesn't actually support your specific GPU silicon (INF mod isn't deep enough). This is a wall — no easy fix from VM side.
3. ReBAR mismatch. Check `lspci -vvv -s 01:00.0 | grep "Region 1"` on the host. Try resizing BAR1 on the host before VM start.

### `BlInitializeLibrary failed 0xc0000225` during boot

→ UEFI/CSM mismatch. Stay on legacy BIOS (SeaBIOS) for Win7. Don't try OVMF unless you really want to deal with it.

### BSOD `0x000000F7 DRIVER_OVERRAN_STACK_BUFFER`

→ Usually a bad driver from a heavily-modified install ISO. Use vanilla Win7 SP1 ISO + UpdatePack7R2 instead.

### "No bootable device" in OVMF/SeaBIOS

→ Boot order. `<boot dev="cdrom"/>` first when installing, `<boot dev="hd"/>` first after.

## Files

- `win7.xml` — Full working libvirt domain XML

## Credits

- The Code 12 / pcie.0 fix comes from [this Proxmox forum thread](https://forum.proxmox.com/threads/gpu-passthrough-windows-7-error-code-12.49474/) by user `dcsapak`. Without it, Win7 q35 GPU passthrough doesn't work.
- UpdatePack7R2 by Simplix
- The Win-Raid community for ongoing Win7 driver work

## License

CC0 / public domain. Do whatever.

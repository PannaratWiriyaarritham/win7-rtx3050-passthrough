# Win7 + RTX 3050 Mobile passthrough on Fedora 43

Working KVM/libvirt config for Windows 7 Ultimate x64 with NVIDIA RTX 3050 6GB Mobile (GA107BM, device ID `10de:25ac`) passthrough on an Acer Nitro ANV15-41 laptop.

## Hardware

- Laptop: Acer Nitro ANV15-41
- CPU: AMD Ryzen 5 6600H (Zen 3+)
- GPU passed through: NVIDIA RTX 3050 6GB Mobile (Ampere, GA107BM)
- Host display: AMD Radeon 660M iGPU
- Host OS: Fedora Linux 43 (kernel 6.19, KDE Plasma)

## Key fixes

### 1. GPU on pcie.0 slot 0x10 multifunction (Code 12 fix)

Win7 q35 has PCIe bridge enumeration bugs. Placing the GPU and audio function directly on pcie.0 root bus as multifunction (slot 0x10, functions 0x0 and 0x1) bypasses this. Credit: [Proxmox forum thread](https://forum.proxmox.com/threads/gpu-passthrough-windows-7-error-code-12.49474/).

### 2. MMIO window expansion

```xml
<qemu:commandline>
  <qemu:arg value="-global"/>
  <qemu:arg value="q35-pcihost.pci-hole64-size=512G"/>
</qemu:commandline>
```

Required for RTX 30-series ReBAR-enabled GPUs.

### 3. Anti-VM detection

```xml
<hyperv>
  <vendor_id state="on" value="randomid"/>
</hyperv>
<kvm>
  <hidden state="on"/>
</kvm>
```

Prevents NVIDIA driver Code 43.

### 4. NVIDIA Win7 driver

INF-modded 475.14 (last official Win7 NVIDIA driver) to add device ID `25AC`.

## Files

- `win7.xml` — Full libvirt domain XML

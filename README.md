# VirtIO-GPU Display Driver Build Guide

## Overview
This guide explains how to build the **VirtIO-GPU (viogpu) display driver** for Windows KVM virtual machines with 3D acceleration support. This driver enables hardware-accelerated graphics in Windows guests running on KVM/QEMU with virtio-gpu.

## What This Driver Does
- Provides **3D acceleration** for Windows guests in KVM/QEMU
- Enables **DirectX support** through virtio-gpu device emulation
- Improves graphics performance for Windows VMs on Linux hosts

## Prerequisites

### Required Software
1. **Windows Driver Kit (WDK) 10.0.26100.0 or later**
   - Download from: https://docs.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk
   - Includes the Windows SDK

2. **Visual Studio 2022** (version 17.14.5 or later)
   - Download from: https://visualstudio.microsoft.com/downloads/
   - Required components:
     - Desktop development with C++
     - Windows Driver Kit integration

3. **Enterprise Windows Driver Kit (EWDK)** (Alternative to VS + WDK)
   - Download from: https://docs.microsoft.com/en-us/legal/windows/hardware/enterprise-wdk-license-2022
   - Self-contained build environment (recommended for CI/CD)

4. **Git** (for cloning the repository)

### System Requirements
- **Operating System**: Windows 10/11 (ARM64 or x64)
- **Disk Space**: ~15 GB for tools + source code
- **Architecture**: x64 (for building x64 drivers) or ARM64 (for ARM64 drivers)

## Build Instructions

### Step 1: Clone the Repository
```cmd
git clone https://github.com/virtio-win/kvm-guest-drivers-windows.git
cd kvm-guest-drivers-windows
```

### Step 2: Set Up Build Environment
Open **"x64 Native Tools Command Prompt for VS 2022"** (or ARM64 variant for ARM builds)

Alternatively, if using EWDK:
```cmd
cd C:\\EWDK
LaunchBuildEnv.cmd
```

### Step 3: Build VirtioLib Dependency
The viogpu driver requires VirtioLib to be built first:

```cmd
cd VirtIO
msbuild VirtioLib.vcxproj /p:Configuration="Win11 Release" /p:Platform=x64
```

**For Win10 or different architectures:**
- Win10: `/p:Configuration="Win10 Release"`
- ARM64: `/p:Platform=ARM64`

### Step 4: Build VirtIO-GPU Driver
```cmd
cd ..\\viogpu
msbuild viogpu.sln /p:Configuration="Win11 Release" /p:Platform=x64
```

### Step 5: Locate Build Output
The compiled driver files are located in:
```
viogpu\\viogpudo\\objfre_win11_amd64\\amd64\\
```

For other configurations:
- **Win10 x64**: `objfre_win10_amd64\\amd64\\`
- **Win11 ARM64**: `objfre_win11_arm64\\arm64\\`
- **Win10 ARM64**: `objfre_win10_arm64\\arm64\\`

## Build Output Files

### Essential Driver Files
| File | Description |
|------|-------------|
| `viogpudo.sys` | Kernel-mode display driver |
| `viogpudo.inf` | Driver installation information |
| `viogpudo.cat` | Driver catalog file (for signing) |
| `viogpuap.exe` | User-mode application component |
| `viogpusrv.exe` | User-mode service component |
| `*.pdb` | Debug symbol files (optional) |

## Installation in Windows VM

### Method 1: Manual Installation
1. **Copy** the entire `amd64` (or `arm64`) folder to your Windows VM
2. **Right-click** `viogpudo.inf` → Select **"Install"**
3. **Reboot** the VM

### Method 2: Device Manager Installation
1. Open **Device Manager**
2. Locate the display adapter (may show as "Standard VGA" or "Microsoft Basic Display Adapter")
3. Right-click → **Update driver** → **Browse my computer**
4. Point to the folder containing the driver files
5. **Reboot** the VM

### Method 3: Automated Installation
```cmd
pnputil /add-driver viogpudo.inf /install
```

## Troubleshooting Build Errors

### Error: Missing bcryptprovider.h
**Cause**: Building the `viocrypt` driver which requires Cryptographic Provider Development Kit

**Solution**: Build only viogpu (as shown above) or skip viocrypt in the solution configuration

### Error: Missing winfsp/winfsp.h
**Cause**: Building the `virtiofs` driver which requires WinFsp

**Solution**: 
- Install WinFsp from https://winfsp.dev/ or
- Build only viogpu to avoid this dependency

### Error: Cannot open input file 'virtiolib.lib'
**Cause**: VirtioLib dependency not built

**Solution**: Build VirtioLib first (see Step 3)

## Target Platforms

### Supported Windows Versions
- Windows 10 (1809 and later)
- Windows 11
- Windows Server 2019/2022

### Supported Architectures
- **x64** (AMD64/Intel 64-bit)
- **ARM64** (AArch64)

## KVM/QEMU Configuration

To use this driver, ensure your VM is configured with virtio-gpu:

### QEMU Command Line
```bash
-device virtio-gpu-pci,id=video0
```

### libvirt XML
```xml
<video>
  <model type='virtio' heads='1' primary='yes'>
    <acceleration accel3d='yes'/>
  </model>
</video>
```

## License
This driver is part of the virtio-win project and follows the same licensing terms.

## Additional Resources
- **Project Repository**: https://github.com/virtio-win/kvm-guest-drivers-windows
- **KVM Documentation**: https://www.linux-kvm.org/
- **Windows Driver Documentation**: https://docs.microsoft.com/en-us/windows-hardware/drivers/

## Build Date
Built on: December 1, 2025

## Notes
- Driver signing is required for production use on Windows 10/11
- Test mode can be enabled for development: `bcdedit /set testsigning on`
- For signed drivers, use Windows Hardware Lab Kit (HLK) for certification

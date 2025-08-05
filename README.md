# NVIDIA 580.65.06 Regression on Arch Linux: Driver Fails to Load at Boot (No DRM Hand-off)

This repository documents a **regression introduced in NVIDIA driver 580.65.06** on Arch Linux that causes the kernel module to fail initializing properly during early boot. This leads to the NVIDIA driver not exposing GPU information, breaking graphical environments like Hyprland. One noticeable symptom is that **EDID fails to populate**, resulting in `Unknown-1` monitor labels — but the deeper issue is the **missing DRM/KMS hand-off**, which breaks the driver.

## 🧪 Affected Environment

- **GPU**: NVIDIA GeForce RTX 4090 [Discrete] Integrated GPU Disabled via BIOS (AUR driver tested: `nvidia-beta-dkms, nvidia-utils-beta, lib32-nvidia-utils-beta`)
- **Display**: Dell AW3821DW (3840x1600@120Hz)
- **Compositor**: Hyprland 0.50.1 (Wayland)
- **Kernel**: Arch Linux (Linux 6.15.9-arch1-1)
- **Boot**: systemd-boot, with early module loading for `nvidia`, `nvidia_modeset`, `nvidia_uvm`, `nvidia_drm`

---

## 🔎 How the Issue Was Discovered

With NVIDIA driver `580.65.06`, the display resolution in tty and Hyprland was incorrect and `hyprctl` reported the monitor as:

```bash
user@workstation~ % hyprctl monitors all
Monitor Unknown-1 (ID 0):
        1024x768@59.99900 at 0x0
        description:
        make:
        model:
        serial:
        active workspace: 1 (1)
        special workspace: 0 ()
        reserved: 0 30 0 0
        scale: 1.00
        transform: 0
        focused: yes
        dpmsStatus: 1
        vrr: false
        solitary: 0
        activelyTearing: false
        directScanoutTo: 0
        disabled: false
        currentFormat: XRGB8888
        mirrorOf: none
        availableModes: 1024x768@60.00Hz
```

…and `nvidia-smi` showed **no running processes**, even though Hyprland and Xwayland were active:

```bash
user@workstation ~ % nvidia-smi
Mon Aug  4 23:59:57 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.65.06              Driver Version: 580.65.06      CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4090        Off |   00000000:01:00.0 Off |                  Off |
| 31%   35C    P0             58W /  600W |       1MiB /  24564MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

This indicated a driver-level failure — not just an EDID issue.

## ✅ Working Version: 575.64.05

When reverting to driver 575.64.05 via this method:

```bash
yay -G nvidia-beta-dkms
cd nvidia-beta-dkms
git checkout 8a60c6fbedd132073d427772937204e86267238a
makepkg -si

yay -G nvidia-utils-beta
cd nvidia-utils-beta
git checkout bdf89b1cddb4b13d76520e1e908a838f897be318
makepkg -si

yay -G lib32-nvidia-utils-beta
cd lib32-nvidia-utils-beta
git checkout 6265c883f8dbd63667f22c34907a074e4e36219c
makepkg -si

sudo pacman -U ./nvidia-beta-dkms-575.64.05-1-x86_64.pkg.tar.zst \
                ./nvidia-utils-beta-575.64.05-1-x86_64.pkg.tar.zst \
                ./lib32-nvidia-utils-beta-575.64.05-1-x86_64.pkg.tar.zst
sudo mkinitcpio -P && sudo reboot
```

...the driver loads early as expected:

```bash
user@workstation ~ % nvidia-smi
Mon Aug  4 23:52:26 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 575.64.05              Driver Version: 575.64.05      CUDA Version: 12.9     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4090        Off |   00000000:01:00.0  On |                  Off |
| 30%   34C    P0             51W /  600W |     647MiB /  24564MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            1048      G   Hyprland                                296MiB |
|    0   N/A  N/A            1089      G   /usr/bin/ghostty                        209MiB |
|    0   N/A  N/A            1095      G   Xwayland                                  8MiB |
+-----------------------------------------------------------------------------------------+
user@workstation~ % hyprctl monitors all
Monitor DP-2 (ID 0):
        3840x1600@119.98200 at 0x0
        description: Dell Inc. Dell AW3821DW
        make: Dell Inc.
        model: Dell AW3821DW
        serial: #REDACTED
        active workspace: 1 (1)
        special workspace: 0 ()
        reserved: 0 30 0 0
        scale: 1.00
        transform: 0
        focused: yes
        dpmsStatus: 1
        vrr: true
        solitary: 0
        activelyTearing: false
        directScanoutTo: 0
        disabled: false
        currentFormat: XBGR2101010
        mirrorOf: none
        availableModes: 3840x1600@59.99Hz 3840x1600@144.00Hz 3840x1600@119.98Hz 3840x1600@99.97Hz 3840x1600@84.97Hz 1024x768@60.00Hz 800x600@60.32Hz 640x480@59.94Hz
```

---

## ❌ Broken Version: 580.65.06

In contrast, the newer version after upgrade again shows:

```bash
user@workstation ~ % nvidia-smi
Mon Aug  4 23:59:57 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.65.06              Driver Version: 580.65.06      CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4090        Off |   00000000:01:00.0 Off |                  Off |
| 31%   35C    P0             58W /  600W |       1MiB /  24564MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
user@workstation ~ % hyprctl monitors all
Monitor Unknown-1 (ID 0):
        1024x768@59.99900 at 0x0
        description:
        make:
        model:
        serial:
        active workspace: 1 (1)
        special workspace: 0 ()
        reserved: 0 30 0 0
        scale: 1.00
        transform: 0
        focused: yes
        dpmsStatus: 1
        vrr: false
        solitary: 0
        activelyTearing: false
        directScanoutTo: 0
        disabled: false
        currentFormat: XRGB8888
        mirrorOf: none
        availableModes: 1024x768@60.00Hz
```

## 🔍 Diagnosis: Missing KMS Handshake (`modeset=1`) / DRM Modeset Fails Silently

Boot logs showed no `fbcon` handoff or console initialization when using 580.65.06.

### ✅ 575.64.05 boot log:

```
[    0.282703] fbcon: Taking over console
[    3.807812] nvidia: loading out-of-tree module taints kernel.
[    3.807816] nvidia: module license 'NVIDIA' taints kernel.
[    3.807817] Disabling lock debugging due to kernel taint
[    3.807818] nvidia: module verification failed: signature and/or required key missing - tainting kernel
[    3.807819] nvidia: module license taints kernel.
[    3.990869] nvidia-nvlink: Nvlink Core is being initialized, major device number 240
[    3.992540] nvidia 0000:01:00.0: vgaarb: VGA decodes changed: olddecodes=io+mem,decodes=none:owns=none
[    4.054254] nvidia-modeset: Loading NVIDIA Kernel Mode Setting Driver for UNIX platforms  575.64.05  Fri Jul 18 15:45:08 UTC 2025
[    4.067607] nvidia_uvm: module uses symbols nvUvmInterfaceDisableAccessCntr from proprietary module nvidia, inheriting taint.
[    4.120644] [drm] [nvidia-drm] [GPU ID 0x00000100] Loading driver
[    5.449212] [drm] Initialized nvidia-drm 0.0.0 for 0000:01:00.0 on minor 1
[    5.488446] nvidia 0000:01:00.0: vgaarb: deactivate vga console
[    5.504341] fbcon: nvidia-drmdrmfb (fb0) is primary device
[    5.615360] nvidia 0000:01:00.0: [drm] fb0: nvidia-drmdrmfb frame buffer device
```

### ❌ 580.65.06 boot log (before fix):

```
[    0.284065] fbcon: Taking over console
[    3.785038] nvidia: loading out-of-tree module taints kernel.
[    3.785043] nvidia: module license 'NVIDIA' taints kernel.
[    3.785044] Disabling lock debugging due to kernel taint
[    3.785045] nvidia: module verification failed: signature and/or required key missing - tainting kernel
[    3.785046] nvidia: module license taints kernel.
[    3.973303] nvidia-nvlink: Nvlink Core is being initialized, major device number 240
[    3.976036] nvidia 0000:01:00.0: vgaarb: VGA decodes changed: olddecodes=io+mem,decodes=none:owns=none
[    4.041248] nvidia-modeset: Loading NVIDIA Kernel Mode Setting Driver for UNIX platforms  580.65.06  Sun Jul 27 06:40:17 UTC 2025
[    4.054666] nvidia_uvm: module uses symbols nvUvmInterfaceDisableAccessCntr from proprietary module nvidia, inheriting taint.
[    4.105833] [drm] [nvidia-drm] [GPU ID 0x00000100] Loading driver
[    4.105865] [drm] Initialized nvidia-drm 0.0.0 for 0000:01:00.0 on minor 1
# no fbcon messages — no primary device handed off
```

This suggests the framebuffer (fbdev) handoff failed silently, which prevented KMS from initializing correctly early in the boot process.

> See: [`logs/nvidia-boot-diff.patch`](logs/nvidia-boot-diff.patch)

---

## ✅ Solution: Explicitly Enable DRM Modesetting, i.e. `modeset=1` in `nvidia_drm`

Although `modeset=1` is documented as enabled by default on Arch Linux in the [Hyprland Wiki](https://wiki.hypr.land/Nvidia/#early-kms-modeset-and-fbdev), setting it explicitly in /etc/modprobe.d/nvidia.conf resolves the issue for 580.65.06:

```bash
# /etc/modprobe.d/nvidia.conf
options nvidia_drm modeset=1
```

Then:

```bash
sudo mkinitcpio -P
sudo reboot now
```

### ✅ 580.65.06 boot log (after fix)

```
[    0.284261] fbcon: Taking over console
[    3.791734] nvidia: loading out-of-tree module taints kernel.
[    3.791738] nvidia: module license 'NVIDIA' taints kernel.
[    3.791738] Disabling lock debugging due to kernel taint
[    3.791739] nvidia: module verification failed: signature and/or required key missing - tainting kernel
[    3.791740] nvidia: module license taints kernel.
[    3.980343] nvidia-nvlink: Nvlink Core is being initialized, major device number 240
[    3.983146] nvidia 0000:01:00.0: vgaarb: VGA decodes changed: olddecodes=io+mem,decodes=none:owns=none
[    4.049538] nvidia-modeset: Loading NVIDIA Kernel Mode Setting Driver for UNIX platforms  580.65.06  Sun Jul 27 06:40:17 UTC 2025
[    4.062949] nvidia_uvm: module uses symbols nvUvmInterfaceDisableAccessCntr from proprietary module nvidia, inheriting taint.
[    4.114121] [drm] [nvidia-drm] [GPU ID 0x00000100] Loading driver
[    5.418511] [drm] Initialized nvidia-drm 0.0.0 for 0000:01:00.0 on minor 1
[    5.457443] nvidia 0000:01:00.0: vgaarb: deactivate vga console
[    5.473046] fbcon: nvidia-drmdrmfb (fb0) is primary device
[    5.589048] nvidia 0000:01:00.0: [drm] fb0: nvidia-drmdrmfb frame buffer device
```

After this change:

- `nvidia-smi` correctly reports GPU processes
- EDID populates
- DRM hand-off succeeds

```bash
user@workstation ~ % nvidia-smi
Tue Aug  5 00:03:20 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.65.06              Driver Version: 580.65.06      CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4090        Off |   00000000:01:00.0  On |                  Off |
| 30%   34C    P0             51W /  600W |    1080MiB /  24564MiB |      1%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            1037      G   Hyprland                                678MiB |
|    0   N/A  N/A            1079      G   /usr/bin/ghostty                        137MiB |
|    0   N/A  N/A            1085      G   Xwayland                                  8MiB |
|    0   N/A  N/A            1223      G   /usr/bin/ghostty                        113MiB |
+-----------------------------------------------------------------------------------------+
user@workstation % hyprctl monitors all
Monitor DP-2 (ID 0):
        3840x1600@119.98200 at 0x0
        description: Dell Inc. Dell AW3821DW
        make: Dell Inc.
        model: Dell AW3821DW
        serial: #REDACTED
        active workspace: 1 (1)
        special workspace: 0 ()
        reserved: 0 30 0 0
        scale: 1.00
        transform: 0
        focused: yes
        dpmsStatus: 1
        vrr: true
        solitary: 0
        activelyTearing: false
        directScanoutTo: 0
        disabled: false
        currentFormat: XBGR2101010
        mirrorOf: none
        availableModes: 3840x1600@59.99Hz 3840x1600@144.00Hz 3840x1600@119.98Hz 3840x1600@99.97Hz 3840x1600@84.97Hz 1024x768@60.00Hz 800x600@60.32Hz 640x480@59.94Hz
```

---

## 🔁 Steps to Reproduce

1. Install NVIDIA 580.65.06
2. Reboot without `modeset=1` explicitly set
3. Observe: `nvidia-smi` shows no processes, `hyprctl monitors` shows `Unknown-1`
4. Downgrade to 575.64.05 or re-enable `modeset=1` manually
5. Confirm: module loads and all components behave normally

---

## 📁 Files Included

- `logs/nvidia-575-boot.log`: Working boot with EDID exposed
- `logs/nvidia-580-boot.log`: Broken boot without fbcon handoff
- `logs/nvidia-580-modeset-boot-log.txt`: Working 580.65.06 with `modeset=1` explicitly set
- `logs/nvidia-boot-diff.patch`: Unified diff of 575 vs 580
- `logs/nvidia-boot-diff-modeset.patch`: Unified diff of 580 before/after `modeset=1`
- `config/mkinitcpio.conf`: Example early module load configuration

---

## 📝 Recommendation

This is **not an EDID bug** — it’s a **regression in driver module initialization** in version 580.65.06. While the `modeset=1` option is reportedly enabled by default in Arch's NVIDIA packaging, it appears the driver _silently ignores_ this unless explicitly set in `/etc/modprobe.d`.

NVIDIA should investigate why the driver no longer initializes `fbcon` or fully activates GPU processes without this manual flag.

A patch or at minimum a doc clarification in Hyprland's Wiki may be warranted.

> If you're an Arch or Hyprland user encountering `Unknown-1` monitors after updating your NVIDIA driver: **enable `modeset=1`.**

---

## 🙋‍♂️ Maintainer

@PixelHabits

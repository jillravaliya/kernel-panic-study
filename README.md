# A Single Missing File Took Down the Entire Boot

![Linux](https://img.shields.io/badge/LINUX-000000?style=for-the-badge&logo=linux&logoColor=FCC624)
![Ubuntu](https://img.shields.io/badge/UBUNTU_24.04-000000?style=for-the-badge&logo=ubuntu&logoColor=E95420)
![Kernel](https://img.shields.io/badge/KERNEL_6.17.0--14-000000?style=for-the-badge&logo=gnu&logoColor=0078D4)
![Bug](https://img.shields.io/badge/BUG_%232141741-000000?style=for-the-badge&logo=hackaday&logoColor=00D26A)
![Open Source](https://img.shields.io/badge/OPEN_SOURCE_CONTRIBUTION-000000?style=for-the-badge&logo=git&logoColor=E95420)

> **Kernel panic on boot. Storage fine. GPU fine. The system simply never got to see the disk — because a missing initrd file went completely unnoticed during installation.**

---

## What Happened

On **February 13, 2026**, the laptop booted into a kernel panic instead of the login screen:

```
KERNEL PANIC!
VFS: Unable to mount root fs on unknown-block(0,0)
```

- Kernel `6.17.0-14` had been installed three days earlier via `apt upgrade`
- The initrd file for `6.17` was **never generated** during installation
- GRUB created a boot entry pointing to a file that did not exist
- No error shown. No warning. The system acted as if installation had succeeded.

> Booting the previous kernel (`6.14`) worked perfectly. The panic was specific to `6.17`.

---

## Root Cause

### The Short Version

> A VirtualBox DKMS compilation failure blocked the initrd generation step during kernel installation. A systemd post-install script then detected the missing initrd — and silently returned exit code 0 (success) anyway. GRUB created an unbootable boot entry with no warning to the user.

---

### The Full Picture

When Ubuntu installs a new kernel, post-install scripts run automatically from `/etc/kernel/postinst.d/`. These handle:

- Compiling third-party kernel modules via DKMS
- Generating the initrd via `initramfs-tools`
- Updating the GRUB bootloader configuration

`run-parts` — the program that executes these scripts — runs them in **alphabetical order** and **stops immediately if any script returns a non-zero exit code**:

```
dkms                  ← runs first  (d)
initramfs-tools       ← runs second (i)
zz-update-grub        ← runs last   (z)
```

DKMS (Dynamic Kernel Module Support) automatically recompiles third-party kernel modules — like VirtualBox drivers, NVIDIA drivers, or VPN modules — whenever a new kernel is installed. On this system, VirtualBox `7.0.16-dfsg` was installed. The `-dfsg` suffix means Ubuntu stripped some proprietary components from the package to comply with open-source guidelines — but this also removed header files the DKMS module needs to compile.

The compilation failed with **exit code 11.**

> `run-parts` saw the non-zero exit from `dkms` and stopped. `initramfs-tools` — the script responsible for generating the initrd — **never ran.**

---

### Why NVMe Systems Panic Without initrd

The initrd (initial RAM disk) is a small temporary filesystem GRUB loads into RAM alongside the kernel at boot. It contains the storage drivers the kernel needs to access the disk — including the NVMe driver.

The kernel configuration on this system:

```
CONFIG_BLK_DEV_NVME=m   ← NVMe driver is a loadable module, not built into the kernel
CONFIG_ATA=y             ← SATA driver IS built directly into the kernel
```

- `=m` means the driver is a **separate file** that must be loaded at runtime — it lives inside the initrd
- `=y` means the driver is compiled directly into the kernel binary — no initrd needed

Without initrd, the NVMe module never loads. The kernel boots with no way to see the SSD. `unknown-block(0,0)` — no device found — kernel panic.

> SATA systems are unaffected because `CONFIG_ATA=y` — the driver is already inside the kernel itself.

---

### The systemd Silent Failure

After the initrd went missing, a script from the systemd package ran — `55-initrd.install`. Its job is to copy the initrd to the correct boot location. It checks whether the initrd exists:

```bash
if [ -e "$INITRD_SRC" ]; then
    install -m 0644 "$INITRD_SRC" "$INITRD_DEST"
else
    [ "$KERNEL_INSTALL_VERBOSE" -gt 0 ] && \
      echo "$INITRD_SRC does not exist, not installing an initrd"
fi

exit 0
```

What happened, step by step:

- Script checked for initrd — **not found**
- Verbose mode is off by default during `apt upgrade` — **nothing printed**
- Script returned **exit 0 — success**

> GRUB received a success signal. Confidently created a boot entry for `6.17` pointing to a file that did not exist. The user saw a completed upgrade with no indication anything was wrong.

The behavior suggests the correct response would be **exit 1** — causing dpkg to report a visible error and preventing GRUB from writing an unbootable entry. That is what the bug report covers.

---

### Why the Same Failure Repeated Three Days in a Row

The dpkg log — the record of every package operation on the system — shows the identical failure on three consecutive days:

```
2026-02-11 09:09:32  status half-configured linux-image-6.17.0-14-generic
2026-02-12 09:36:03  status half-configured linux-image-6.17.0-14-generic
2026-02-13 08:41:13  status half-configured linux-image-6.17.0-14-generic
```

- Each day `apt upgrade` retried the pending kernel configuration
- Each day it hit the same DKMS failure
- Each day it silently left the system in a broken `half-configured` state

> On February 13, the laptop was rebooted for the first time since kernel `6.17` was installed. The panic appeared immediately.

---

### Trigger vs Underlying Issue

> VirtualBox's broken DKMS package was the **trigger**. The **underlying issue** is that systemd's install script allows the process to complete silently when a critical boot file is missing.

Any DKMS module failure would produce the same outcome on an NVMe system:

- NVIDIA driver fails to compile
- ZFS module fails to compile
- WireGuard or any custom driver fails to compile

All of them would block `initramfs-tools` through the same path. `55-initrd.install` would silently succeed in every case. VirtualBox was just what happened to be installed here.

---

## The Fix Applied

Removed VirtualBox to clear the DKMS failure, then regenerated the initrd manually:

```bash
$ sudo apt remove --purge virtualbox
$ sudo update-initramfs -c -k 6.17.0-14-generic
$ sudo update-grub
$ sudo reboot
```

**Verify:**

```bash
$ uname -r
6.17.0-14-generic

$ ls -la /boot/initrd*
-rw-r--r--  73101720  initrd.img-6.14.0-37-generic
-rw-r--r--  73711966  initrd.img-6.17.0-14-generic
```

> Both kernels have valid initrd files. System stable.

---

## Bug Report

> **Reported to:** systemd package / Ubuntu kernel team
**[Launchpad #2141741 — 55-initrd.install silently exits 0 when initrd missing, causing undetectable kernel panic on NVMe systems](https://bugs.launchpad.net/ubuntu/+source/systemd/+bug/2141741)**

- Status: **Confirmed** — within 2 hours of filing



**Observed behavior:**

- DKMS failure stops `run-parts` before `initramfs-tools` runs
- `55-initrd.install` detects missing initrd and returns exit 0 instead of exit 1
- GRUB creates unbootable boot entry with no visible error
- NVMe systems panic on next reboot — SATA systems unaffected

**Evidence attached to report:**

- Full `dpkg.log` output — three consecutive `half-configured` states across Feb 11–13
- Full `apt/term.log` — DKMS exit code 11 and VirtualBox compilation error
- Contents of `55-initrd.install` showing the `exit 0` on missing initrd
- `ls /boot/initrd*` output confirming the missing file
- `cat /boot/config-6.17.0-14-generic` showing `CONFIG_BLK_DEV_NVME=m`

> This connects to **[Bug #2136499](https://bugs.launchpad.net/ubuntu/+bug/2136499)** — the VirtualBox DKMS build failure affecting **110+ users**. The systemd silent exit is what turns that DKMS failure into an unbootable system.

---

## Affected Systems

- Any system with NVMe storage where `CONFIG_BLK_DEV_NVME=m`
- Any DKMS module installed — VirtualBox, NVIDIA, ZFS, WireGuard, and others
- Ubuntu `24.04` / kernel `6.17+`

---

## Repo Structure

```
README.md          ← You are here
analysis.md        ← Full investigation: every step, every log, complete failure chain
screenshots/       ← Supporting images referenced in analysis.md
```

> Full investigation with every step and the complete failure chain: **[analysis.md](./analysis.md)**

---

## Connect With Me

I'm actively learning and building in the **systems programming** and **Linux** space.

- **Email:** jillahir9999@gmail.com
- **LinkedIn:** [linkedin.com/in/jill-ravaliya-684a98264](https://linkedin.com/in/jill-ravaliya-684a98264)
- **GitHub:** [github.com/jillravaliya](https://github.com/jillravaliya)

**Open to:**

- Systems programming mentorship
- Linux and open source contribution guidance
- Technical discussions on boot infrastructure and package management
- Collaboration on systems-level projects

---

> *Three days of silent failures. One missing file. One script returning success when it should not have. The system never warned me — so I had to read everything until it did.*

<div align="center">
<img src="https://readme-typing-svg.herokuapp.com?font=Fira+Code&size=18&pause=800&duration=3000&color=0078D4&center=true&vCenter=true&width=600&lines=From+Panic+to+Report;systemd+Bug+Discovered;Confirmed+in+2+Hours;Silent+Failures+Exposed;NVMe+Systems+Protected" alt="Typing SVG" />
</div>

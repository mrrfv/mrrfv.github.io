---
title: How to install NVIDIA drivers on Fedora Silverblue 44 (with Secure Boot)
date: 2025-12-29 06:50:53 +0100
last_modified_at: 2026-05-17 06:50:53 +0100
categories: [Linux]
tags: [linux, nvidia]
---

Setting up NVIDIA drivers on an atomic desktop like Fedora Silverblue can be daunting, especially with Secure Boot enabled. In reality, it's a straightforward, one-time process. *Originally published for version 43, now updated.*

## Overview

Since Fedora Silverblue is an atomic system with a read-only core, installing NVIDIA drivers involves "layering" them on top of the base OS using `rpm-ostree`. Secure Boot blocks any hardware driver it doesn't recognize, meaning you must create a digital signature and register it with your computer's BIOS. Once this trust is established, the system can automatically sign the proprietary NVIDIA driver during installation and load it.

## Guide

### 1. Update your system
Before layering packages, ensure your base system is up to date. We want to be running the latest kernel.

```bash
sudo rpm-ostree upgrade
# If updates were applied, reboot now:
systemctl reboot
```

### 2. Enable RPM Fusion
Fedora does not ship proprietary drivers, therefore we need to enable the RPM Fusion repositories (both free and non-free).

```bash
sudo rpm-ostree install \
    https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
    https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

> *Command sourced from the [RPM Fusion Configuration Guide](https://rpmfusion.org/Configuration)*

### 3. Install build tools
To compile the NVIDIA kernel modules locally, we need specific build tools and the signing utilities for Secure Boot.

```bash
sudo rpm-ostree install kmodtool akmods rpmdevtools mock
# Fedora Silverblue 43 users: add openssl to the end of this command.
```
**Important:** You need to reboot after this step, unless you use `--apply-live`. Otherwise the commands below won't work, as the tools aren't present in the current environment.

### 4. Generate and import a Secure Boot key
We must create a cryptographic key to sign the drivers, then import it into shim.

**Generate the key:**
```bash
sudo kmodgenca -a
```

**Import it:**
```bash
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
```
*You will be asked to create a temporary password. Remember this! You will need it in the next step.*

> *Process courtesy of [roworu](https://github.com/roworu/nvidia-fedora-secureboot)*

### 5. Install silverblue-akmods-keys

According to the [RPM Fusion docs](https://rpmfusion.org/Howto/NVIDIA#OSTree_.28Silverblue.2FKinoite.2Fetc.29):

> There is a need for a special "Hack" to work with secureboot on OSTree systems. Once you have the keys generated and imported, you need to create a dedicated packages [sic] for the key to be available when signing the kernel modules in OStree.

We'll install `silverblue-akmods-keys` to provide this package:

```bash
# Clone CheariX/silverblue-akmods-keys
git clone https://github.com/CheariX/silverblue-akmods-keys
cd silverblue-akmods-keys
# Build and install the package
sudo bash setup.sh
rpm-ostree install akmods-keys-0.0.2-8.fc$(rpm -E %fedora).noarch.rpm
```

### 6. Reboot & enroll key
Reboot your computer now.

```bash
systemctl reboot
```

Upon booting, you will see a blue screen labeled **"Shim UEFI key management"**. [Tenk fort](https://youtu.be/IjU_v6SWvDU):

1.  Press any key to enter the menu.
2.  Select **Enroll MOK**.
3.  Select **Continue**.
4.  Select **Yes**.
5.  Enter the **password** you created in Step 4.
6.  Select **Reboot**.

If you miss this screen, the key won't be trusted, and the driver will fail to load.

### 7. Install the NVIDIA drivers
Now that our signing key is enrolled, we can layer the actual drivers. This command installs the driver, CUDA libs, and hardware acceleration support.

```bash
sudo rpm-ostree install akmod-nvidia xorg-x11-drv-nvidia xorg-x11-drv-nvidia-cuda xorg-x11-drv-nvidia-cuda-libs libva-nvidia-driver.{i686,x86_64} xorg-x11-drv-nvidia-libs.{i686,x86_64}
```

It's expected that this command takes some time to finish.

We also need to modify kernel arguments to prevent the open-source `nouveau` driver from conflicting with NVIDIA.

```bash
sudo rpm-ostree kargs --append=rd.driver.blacklist=nouveau --append=modprobe.blacklist=nouveau --append=rd.driver.blacklist=nova_core --append=modprobe.blacklist=nova_core
```

> *Sourced from [RPM Fusion NVIDIA HOWTO](https://rpmfusion.org/Howto/NVIDIA#OSTree_.28Silverblue.2FKinoite.2Fetc.29)*

### 8. Final reboot
Reboot your system one last time to boot into the new deployment with the drivers installed.

Once logged in, verify the installation:

```bash
modinfo -F version nvidia
# Should return the driver version (e.g., 565.xx)
```

Have fun gaming!

### Troubleshooting
If `nvidia-smi` returns an error, check if the key was enrolled correctly:
```bash
mokutil --list-enrolled
```
If you don't see your key (often labeled with your username/hostname), repeat Step 4 and 5.

You could also try reinstalling the drivers (step 6) and waiting at least 5 minutes before rebooting. Sometimes the kernel modules compile in the background.

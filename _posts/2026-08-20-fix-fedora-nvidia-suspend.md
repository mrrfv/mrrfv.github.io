---
title: Fix Fedora Workstation 44 not going to sleep with NVIDIA open drivers
date: 2026-08-20 10:00:00 +0100
categories: [Linux]
tags: [linux, nvidia]
---

You leave your computer unattended for more than a few hours. Perhaps you wanted to go outside. Either way, you return to the sound of your computer's fans humming, and everything else appearing nonfunctional: the keyboard doesn't respond to the Caps Lock key, and there's no display output, either!

What happened? NVIDIA's the culprit. Or SELinux, depending on your portfolio :)

## The Problem

The NVIDIA driver on Fedora tries to save the contents of the GPU's video memory to `/var/tmp` at suspend, in order to prevent application crashes and graphical glitches when resuming ([source](https://github.com/rpmfusion/nvidia-kmod/blob/4f29c85d196da4a91a671e2d5272e5eb748ba5ad/set_driver_defaults.patch#L21-L22)). However, because Fedora's SELinux policy doesn't allow for that (see [this GitHub issue](https://github.com/fedora-selinux/selinux-policy/pull/3087)), the driver panics, and takes everything graphics related with it:

```
Aug 19 21:36:22 fedora systemd-sleep[7092]: Performing sleep operation 'suspend'...
...
Aug 19 21:36:22 fedora audit[7092]: AVC avc:  denied  { write open } for  pid=7092 comm="systemd-sleep" path=2F7661722F746D702F23353931383139202864656C6574656429 dev="dm-0" ino=591819 scontext=system_u:system_r:systemd_sleep_t:s0 tcontext=system_u:object_r:tmp_t:s0 tclass=file permissive=0
...
Aug 19 21:36:22 fedora kernel: NVRM: nvAssertFailedNoLog: Assertion failed: listCount(&pKernelBus->virtualBar2[gfid].usedMapList) == 0 @ kern_bus_vbar2.c:346
Aug 19 21:36:23 fedora kernel: NVRM: kgspHealthCheck_TU102: ****************************** GSP-CrashCat Report *******************************
Aug 19 21:36:23 fedora kernel: NVRM: kgspPrintGspBinBuildId_IMPL: GSP bin buildId: 4f09703c5c7d57baa527d6a189bb6d118bc60e49
Aug 19 21:36:23 fedora kernel: NVRM: Xid (PCI:0000:23:00): 120, GSP task exception: load access page fault (cause:0xd) @ pc:0x13636b2, partition:2#0, task:3, gfid: 0
Aug 19 21:36:23 fedora kernel: NVRM:     Reported by libos partition:2#4 kernel v3.1 [0] @ ts:1787168183
...
...
Aug 19 21:37:23 fedora com.brave.Browser.desktop[6033]: [55:55:0819/213723.221198:ERROR:gpu/command_buffer/service/shared_context_state.cc:1317] SharedContextState context lost via EXT_robustness. Reset status = GL_GUILTY_CONTEXT_RESET_KHR
Aug 19 21:37:23 fedora com.brave.Browser.desktop[6033]: [55:55:0819/213723.221614:ERROR:components/viz/service/gl/exit_code.cc:13] Restarting GPU process due to unrecoverable error. Context was lost.
Aug 19 21:37:23 fedora com.brave.Browser.desktop[5455]: [2:2:0819/213723.248184:ERROR:content/browser/gpu/gpu_process_host.cc:1035] GPU process exited unexpectedly: exit_code=8704
```

*Note: `path=2F7661722F746D702F23353931383139202864656C6574656429` decodes to `/var/tmp/#591819 (deleted)`.*

Fixing this takes about 15 minutes. We need to capture the logs from the crash as well as details of the SELinux denial, and create a new module allowing the write.

## Analysis

Don't skip this part. It determines if my proposed fix will even work for you at all.

### 1. Enable and start SSH

The patient we're dealing with is actually still completely functional, sans display output. Therefore, if we enable SSH, we can log in remotely and extract diagnostic messages from it, seeing exactly what happened.

Open the Terminal and enable the SSH service:

```bash
sudo systemctl enable --now sshd.service
```

`--now` starts it *now*, in addition to enabling it at boot.

Take note of your computer's IP address, e.g., with `ip a`.

### 2. Suspend

Suspend the machine. We want to induce the crash, but this time, we have access after it'd happened.

### 3. Log in to the computer

Log in to the affected computer using SSH. On a phone, you can use [ConnectBot](https://connectbot.org/) (Android) or [Termius](https://termius.com/) (cross-platform).

Or, on any computer:

```bash
ssh YOUR_LINUX_USERNAME@YOUR_COMPUTER_IP
```

Replace the placeholders with the appropriate information, without adding spaces in between them. You also must be on the same network for the connection to succeed.

### 4. Save the current system logs to a file

Run:

```bash
journalctl -b > nvidia_crash_logs.txt
```

### 5. Search for AVC denial messages in the logs

```bash
grep 'avc' nvidia_crash_logs.txt
```

If you see something like this:

```
Aug 19 21:36:22 fedora audit[7092]: AVC avc:  denied  { write open } for  pid=7092 comm="systemd-sleep" path=2F7661722F746D702F23353931383139202864656C6574656429 dev="dm-0" ino=591819 scontext=system_u:system_r:systemd_sleep_t:s0 tcontext=system_u:object_r:tmp_t:s0 tclass=file permissive=0
```

Then you're experiencing the same exact problem! Note down the numbers in the brackets after `audit` (in the example above, it's 7092), and carry on. To be precise, it's the PID we're looking for.

If not, there's one more thing you could try. Create a new file `/etc/modprobe.d/nvidia-power-management.conf` as root, and add the following contents:

```
options nvidia NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=/var/tmp NVreg_UseKernelSuspendNotifiers=1
```

According to the [NVIDIA source code](https://github.com/NVIDIA/open-gpu-kernel-modules/blob/e4a5faa2567f28c8eabe0ebb6422b6d0abcf37eb/kernel-open/nvidia/nv-reg.h), here's an explanation of the options:

- `UseKernelSuspendNotifiers`: *If enabled, this option prompts the NVIDIA kernel module to register a notifier that saves and restores all video memory allocations across system power management cycles if PreserveVideoMemoryAllocations is enabled.*
- `TemporaryFilePath`: *When specified, this option changes the location in which the NVIDIA kernel module will create unnamed temporary files (e.g. to save the contents of video memory in).  The indicated file must be a directory.  By default, temporary files are created in /tmp.*
- `PreserveVideoMemoryAllocations`: *If enabled, this option prompts the NVIDIA kernel module to save and restore all video memory allocations across system power management cycles, i.e. suspend/resume and hibernate/restore.  Otherwise, only select allocations are preserved.*

Keep in mind that the above options enable the same features that crash on vanilla Fedora with SELinux enabled. Don't fret if it doesn't start working on the first try, but see if this guide becomes relevant.

## Fix

Now that you've diagnosed what's happening to the PC, it's time to create a SELinux module (rule) that grants the NVIDIA driver write access to `/var/tmp`.

Run the following command, *replacing `7092` with the number you jotted down*:

```bash
sudo ausearch -m avc -p 7092 | audit2allow -M systemd_sleep_tmp
```

If the above doesn't work, try a broader search, without the PID number:

```bash
grep 'avc' nvidia_crash_logs.txt | audit2allow -M systemd_sleep_tmp
```

It creates files for the rule, based on the specific denial that occurred. You can optionally review it by running `cat systemd_sleep_tmp.te`.

Install the newly created module with:

```bash
sudo semodule -i systemd_sleep_tmp.pp
```

You might need to reboot for the changes to take effect.

## Further reading

- [Interpreting AVC messages - Red Hat](https://redhatquickcourses.github.io/selinux-policies/selinux-policies/1/ch2-configure/s3-avc.html)
- [NVIDIA suspend fix - GitHub Gist](https://gist.github.com/bmcbm/375f14eaa17f88756b4bdbbebbcfd029)
- [com/NVIDIA/open-gpu-kernel-modules issue: System suspend fails on RTX 4050 Mobile with GSP unload failed 0x62 → kernel WARNING in nv_suspend_devices, leaving system unresponsive (driver 595.71.05, kernel 7.0.4)](https://github.com/NVIDIA/open-gpu-kernel-modules/issues/1142)

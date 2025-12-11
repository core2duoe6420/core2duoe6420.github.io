---
title: "Fixing 'unable to install the EFI boot loader' when installing PVE"
date: 2025-11-30
slug: pve-efi-boot-loader
hide_site_title: true
tags:
- proxmox
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

I recently bought a small J1900 mini PC to give to a friend as a soft router. As usual, I tried to install Proxmox on it, but ran into an issue I’d never seen before: when the progress bar reached 100%, it threw this error: `failed to prepare EFI boot using Grub on '/dev/sda2': unable to install the EFI boot loader on '/dev/sda'`. I then spent the whole afternoon wrestling with it before finally fixing the problem.
<!--more-->

After searching the web and trying various suggestions, none of the following worked:

- According to [this thread](https://forum.proxmox.com/threads/cannot-install-pve-as-uefi-os.126755/), I tried running `rm /sys/firmware/efi/efivars/dump-*` to free up `efivarfs` space, but I found there were no `dump-*` files at all on my machine, and `efivarfs` still had 60% free space, so it was probably unrelated
- Used `wipefs` and `dd` to completely clear the partition table and file systems
- Tweaked various BIOS parameters
- Reset BIOS settings to default
- Opened the mini PC and tried to locate NVRAM-related jumpers, but couldn’t find any (idea inspired by [this thread](https://forum.proxmox.com/threads/proxmox-7-0-failed-to-prepare-efi.95466/))
- Gave up on EFI boot and tried using CSM instead, but CSM couldn’t even boot the installer

After digging into it for quite a while, I could almost conclude that the BIOS of this industrial PC has a bug that causes EFI boot entry writes to fail. In the first thread mentioned above, a guru in reply #15 provided a manual boot installation method: the idea is to use the default EFI boot file `BOOTX64.EFI`, so the system can boot normally even without writing EFI boot entries. However, step 4 of his procedure requires entering Rescue boot, which I simply couldn’t access from the installer, so I got stuck again.

In the end, the problem was solved thanks to a clutch hint from ChatGPT, which suggested using `chroot`. After some trial and error, it worked. Here are the exact steps:

1. First install PVE as usual; it will throw an error at the end. Reboot into the installer again, then press `Ctrl+Shift+F3` to enter the console.
2. Run the following commands to mount the file systems on the hard drive (assuming the default partition layout):

    ```bash
    vgchange -ay
    mount /dev/pve/root /mnt
    mount /dev/sda2 /mnt/boot/efi
    mount --bind /dev /mnt/dev
    mount --bind /proc /mnt/proc
    mount --bind /sys /mnt/sys
    chroot /mnt
    ```

3. Then follow the guru’s steps from the thread:

    ```bash
    cp /boot/efi/EFI/proxmox/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
    cp -r /boot/grub/x86_64-efi /boot/efi/EFI/BOOT/
    grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=proxmox --removable --recheck
    update-grub
    ```

After rebooting, the system can boot normally. Many thanks to the forum guru and to ChatGPT. 🙏
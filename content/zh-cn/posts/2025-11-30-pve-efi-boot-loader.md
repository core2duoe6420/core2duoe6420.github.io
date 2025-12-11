---
title: "解决安装PVE时unable to install the EFI boot loader的问题"
date: 2025-11-30
slug: pve-efi-boot-loader
hide_site_title: true
tags:
- proxmox
---

最近新买了一个J1900小主机打算送给朋友做软路由。到手后照常安装Proxmox，结果遇到了一个以前没有遇到过的问题，在进度条已经到100%的时候，报了这个错：failed to prepare EFI boot using Grub on '/dev/sda2': unable to install the EFI boot loader on '/dev/sda'。然后又是折腾了一下午才解决。
<!--more-->

在网上查阅了各种资料，以下尝试均没有效果：

- 根据[这个帖子](https://forum.proxmox.com/threads/cannot-install-pve-as-uefi-os.126755/)说的，尝试运行`rm /sys/firmware/efi/efivars/dump-*`以腾出`efivarfs`空间，但我发现我的机器上根本没有`dump-*`文件，`efivarfs`也有60%的可用空间，应该与这个无关
- 使用`wipefs`和`dd`彻底清除分区表和文件系统
- 调整各种BIOS参数
- 重置BIOS参数
- 打开小主机，试图找到NVRAM相关的跳线，但是并没有发现（思路来源于[这个帖子](https://forum.proxmox.com/threads/proxmox-7-0-failed-to-prepare-efi.95466/)）
- 放弃EFI引导，干脆使用CSM引导，结果CSM连安装程序都引导不出来

研究了半天，几乎可以断定这个工控机的BIOS有bug，会导致写入EFI引导项失败。那么在上面第一条的那个帖子里，第15楼有个大牛给出了手动安装引导的方案，思路就是使用默认的EFI引导文件`BOOTX64.EFI`，这样不用写入EFI引导项也能正常引导。但是他的步骤中第四步要求进入Rescue boot，但我从安装程序里压根进不去，于是又卡住了。

最后问题能解决得靠ChatGPT神助攻，提到了可以用`chroot`，一番尝试后果真可以。以下是实操步骤：

1. 先正常安装PVE，最后会报错，重启再进安装程序，按`Ctrl+Shift+F3`进入命令行
2. 运行以下命令挂载硬盘上的文件系统（假定是默认的分区）：

    ```bash
    vgchange -ay
    mount /dev/pve/root /mnt
    mount /dev/sda2 /mnt/boot/efi
    mount --bind /dev /mnt/dev
    mount --bind /proc /mnt/proc
    mount --bind /sys /mnt/sys
    chroot /mnt
    ```

3. 然后就可以按照帖子里大牛的步骤来了：

    ```bash
    cp /boot/efi/EFI/proxmox/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
    cp -r /boot/grub/x86_64-efi /boot/efi/EFI/BOOT/
    grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=proxmox --removable --recheck
    update-grub
    ```

重启后就可以正常引导了。感恩论坛大牛和ChatGPT🙏。
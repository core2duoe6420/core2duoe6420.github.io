---
title: "Installing DSM with libvirt"
date: 2022-10-21
slug: DSM-on-libvirt
tags:
- DSM
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

After a recent NAS upgrade, I ended up with two spare 10 TB drives that I didn’t add to the array. I initially planned to use them for an additional ZFS backup, but later found out that zrepl doesn’t support multiple destinations, so I dropped the idea. Since I had just finished setting up WebVirtCloud, I figured I’d try running a black Synology (XPEnology), passthrough the two disks, and get another toy to play with.

<!--more-->

Ever since I installed Ubuntu bare-metal as my NAS, everything else has been fine, except I’ve never been happy with the VM management interface. I previously used a rather awkward [solution](https://github.com/m-bers/docker-virt-manager), which basically uses Broadway to turn the desktop Virtual Machine Manager into a web-based one. It technically works, but the VM display is horribly laggy. WebVirtCloud is a pure web solution written in Python; while it’s nowhere near mature products like Proxmox or Unraid, it’s still a lot better and good enough for now.

Back to the topic. Here are the references I used:

1. [QEMU系列-黑群晖DSM7.0.1](https://www.wnark.com/archives/174.html)
1. [Tutorial: Install/Migrate to DSM 7.x with TinyCore RedPill (TCRP) Loader](https://xpenology.com/forum/topic/62221-tutorial-installmigrate-to-dsm-7x-with-tinycore-redpill-tcrp-loader/)
1. [DSM 7.x Loaders and Platforms](https://xpenology.com/forum/topic/61634-dsm-7x-loaders-and-platforms/)
1. [Tutorial: Install DSM 7.1 on UNRAID 6.10.3](https://xpenology.com/forum/topic/63333-tutorial-install-dsm-71-on-unraid-6103)
1. [群晖DSM7.X版本自动获取SataPortMap和DiskIdxMap的教程](https://wp.gxnas.com/11876.html)

## Environment

The host system is Ubuntu 22.04, but libvirt is installed in a Docker container. The container image is based on Ubuntu 20.04 (because libvirt on 22.04 cannot start VMs due to CGroup2 issues). Therefore, the QEMU version is 4.2.1 and libvirt is 6.0.0.

The black Synology model is DS918+ with DSM version 7.1.1-42962, which is the latest at the time of writing.

Currently, DSM 7 bootloaders are basically all using TinyCore RedPill (TCRP). The TCRP version is ~~0.9.2.8~~ 0.9.2.9.

## Hardware configuration

WebVirtCloud only satisfies the most basic needs for hardware configuration, but fortunately it supports XML config. Most KVM-based virtualization solutions use libvirt XML configs anyway, so I’ll just paste the working configuration I ended up with.

```xml
<domain type='kvm' id='2'>
  <name>DSM</name>
  <uuid>186682f9-0f0b-4a9e-867b-364706339405</uuid>
  <description>None</description>
  <memory unit='KiB'>4194304</memory>
  <currentMemory unit='KiB'>4194304</currentMemory>
  <vcpu placement='static'>2</vcpu>
  <resource>
    <partition>/machine</partition>
  </resource>
  <os>
    <type arch='x86_64' machine='pc-q35-4.2'>hvm</type>
    <loader readonly='yes' type='pflash'>/usr/share/OVMF/OVMF_CODE.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/DSM_VARS.fd</nvram>
    <bootmenu enable='yes'/>
  </os>
  <features>
    <acpi/>
    <apic/>
  </features>
  <cpu mode='host-passthrough' check='none'/>
  <clock offset='utc'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='block' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native'/>
      <source dev='/dev/sdb' index='3'/>
      <backingStore/>
      <target dev='sda' bus='scsi'/>
      <alias name='scsi0-0-0-0'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <disk type='block' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native'/>
      <source dev='/dev/sdc' index='2'/>
      <backingStore/>
      <target dev='sdb' bus='scsi'/>
      <alias name='scsi0-0-0-1'/>
      <address type='drive' controller='0' bus='0' target='0' unit='1'/>
    </disk>
    <disk type='file' device='disk'>
      <driver name='qemu' type='raw' cache='writeback'/>
      <source file='/image/tinycore-redpill-uefi.v0.9.2.8.img' index='1'/>
      <backingStore/>
      <target dev='hdz' bus='usb'/>
      <boot order='1'/>
      <alias name='usb-disk25'/>
      <address type='usb' bus='0' port='3'/>
    </disk>
    <controller type='pci' index='0' model='pcie-root'>
      <alias name='pcie.0'/>
    </controller>
    <controller type='sata' index='0'>
      <alias name='ide'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1f' function='0x2'/>
    </controller>
    <controller type='usb' index='0' model='qemu-xhci'>
      <alias name='usb'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x0'/>
    </controller>
    <controller type='scsi' index='0' model='virtio-scsi'>
      <alias name='scsi0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </controller>
    <interface type='bridge'>
      <mac address='xx:xx:xx:xx:xx:xx'/>
      <source network='host-bridge' portid='90d5d18c-6959-47ee-80b3-ce4774d28600' bridge='br0'/>
      <target dev='vnet0'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <video>
      <model type='vmvga' vram='16384' heads='1' primary='yes'/>
      <alias name='video0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </video>
    <memballoon model='virtio'>
      <alias name='balloon0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </memballoon>    
    <serial type='pty'>
      <source path='/dev/pts/0'/>
      <target type='isa-serial' port='0'>
        <model name='isa-serial'/>
      </target>
      <alias name='serial0'/>
    </serial>
    <console type='pty' tty='/dev/pts/0'>
      <source path='/dev/pts/0'/>
      <target type='serial' port='0'/>
      <alias name='serial0'/>
    </console>
    <input type='mouse' bus='ps2'>
      <alias name='input0'/>
    </input>
    <input type='keyboard' bus='ps2'>
      <alias name='input1'/>
    </input>
    <graphics type='spice' port='5900' autoport='yes' listen='0.0.0.0'>
      <listen type='address' address='0.0.0.0'/>
    </graphics>
    <hub type='usb'>
      <alias name='hub0'/>
      <address type='usb' bus='0' port='1'/>
    </hub>
  </devices>
  <seclabel type='dynamic' model='dac' relabel='yes'>
    <label>+0:+0</label>
    <imagelabel>+0:+0</imagelabel>
  </seclabel>
</domain>
```

The final QEMU command line is as follows:

```shell
/usr/bin/qemu-system-x86_64 \
    -name guest=DSM,debug-threads=on \
    -S \
    -object secret,id=masterKey0,format=raw,file=/var/lib/libvirt/qemu/domain-5-DSM/master-key.aes \
    -blockdev {"driver":"file","filename":"/usr/share/OVMF/OVMF_CODE.fd","node-name":"libvirt-pflash0-storage","auto-read-only":true,"discard":"unmap"} \
    -blockdev {"node-name":"libvirt-pflash0-format","read-only":true,"driver":"raw","file":"libvirt-pflash0-storage"} \
    -blockdev {"driver":"file","filename":"/var/lib/libvirt/qemu/nvram/DSM_VARS.fd","node-name":"libvirt-pflash1-storage","auto-read-only":true,"discard":"unmap"} \
    -blockdev {"node-name":"libvirt-pflash1-format","read-only":false,"driver":"raw","file":"libvirt-pflash1-storage"} \
    -machine pc-q35-4.2,accel=kvm,usb=off,dump-guest-core=off,pflash0=libvirt-pflash0-format,pflash1=libvirt-pflash1-format \
    -cpu host \
    -m 4096 \
    -overcommit mem-lock=off \
    -smp 2,sockets=2,cores=1,threads=1 \
    -uuid 186682f9-0f0b-4a9e-867b-364706339405 \
    -no-user-config \
    -nodefaults \
    -chardev socket,id=charmonitor,fd=44,server,nowait \
    -mon chardev=charmonitor,id=monitor,mode=control \
    -rtc base=utc \
    -no-shutdown \
    -boot menu=on,strict=on \
    -device qemu-xhci,id=usb,bus=pcie.0,addr=0x1 \
    -device virtio-scsi-pci,id=scsi0,bus=pcie.0,addr=0x2 \
    -device usb-hub,id=hub0,bus=usb.0,port=1 \
    -blockdev {"driver":"host_device","filename":"/dev/sdb","aio":"native","node-name":"libvirt-3-storage","cache":{"direct":true,"no-flush":false},"auto-read-only":true,"discard":"unmap"} \
    -blockdev {"node-name":"libvirt-3-format","read-only":false,"cache":{"direct":true,"no-flush":false},"driver":"raw","file":"libvirt-3-storage"} \
    -device scsi-hd,bus=scsi0.0,channel=0,scsi-id=0,lun=0,device_id=drive-scsi0-0-0-0,drive=libvirt-3-format,id=scsi0-0-0-0,write-cache=on \
    -blockdev {"driver":"host_device","filename":"/dev/sdc","aio":"native","node-name":"libvirt-2-storage","cache":{"direct":true,"no-flush":false},"auto-read-only":true,"discard":"unmap"} \
    -blockdev {"node-name":"libvirt-2-format","read-only":false,"cache":{"direct":true,"no-flush":false},"driver":"raw","file":"libvirt-2-storage"} \
    -device scsi-hd,bus=scsi0.0,channel=0,scsi-id=0,lun=1,device_id=drive-scsi0-0-0-1,drive=libvirt-2-format,id=scsi0-0-0-1,write-cache=on \
    -blockdev {"driver":"file","filename":"/image/tinycore-redpill-uefi.v0.9.2.8.img","node-name":"libvirt-1-storage","cache":{"direct":false,"no-flush":false},"auto-read-only":true,"discard":"unmap"} \
    -blockdev {"node-name":"libvirt-1-format","read-only":false,"cache":{"direct":false,"no-flush":false},"driver":"raw","file":"libvirt-1-storage"} \
    -device usb-storage,bus=usb.0,port=3,drive=libvirt-1-format,id=usb-disk25,bootindex=1,removable=off,write-cache=on \
    -netdev tap,fd=46,id=hostnet0,vhost=on,vhostfd=47 \
    -device virtio-net-pci,netdev=hostnet0,id=net0,mac=xx:xx:xx:xx:xx:xx,bus=pcie.0,addr=0x3 \
    -chardev pty,id=charserial0 \
    -device isa-serial,chardev=charserial0,id=serial0 \
    -spice port=5900,addr=0.0.0.0,disable-ticketing,seamless-migration=on \
    -device vmware-svga,id=video0,vgamem_mb=16,bus=pcie.0,addr=0x4 \
    -device virtio-balloon-pci,id=balloon0,bus=pcie.0,addr=0x5 \
    -sandbox on,obsolete=deny,elevateprivileges=deny,spawn=deny,resourcecontrol=deny \
    -msg timestamp=on
```

A brief explanation of this configuration:

- The chipset is Q35 and the BIOS is UEFI, so the TCRP image must also be the UEFI version.
- CPU mode is `host-passthrough`. DS918+ only supports Haswell and newer CPUs; see the reference links for details.
- The USB bus model is `qemu-xhci`, which is USB 3.0; the default is USB 2.0.
- The two passthrough disks are attached to the SCSI bus. SATA would also work, but QEMU’s virtual Q35 SATA controller only supports SATA 1.0 speeds (1.5 Gbps), and in practice read speeds top out at around 150 MiB/s. Moreover, TCRP marks it as Bogus Q35, so better not use it.
- The SCSI controller is `virtio-scsi`, which requires extra drivers, but VirtIO performance is good and can maximize disk throughput.
- The NIC uses `virtio`, which also requires drivers. I tried `e1000e` and `vmxnet3` before that. With `e1000e`, the DS918+ would lose its IP and fail to boot. `vmxnet3` is a 10 Gbps NIC in VMWare, but in my QEMU tests the link speed was only 1 Gbps, and multi-thread tests with iperf3 only reached 600 Mbps. VirtIO performs much better: iperf3 against the host can reach 26 Gbps (my virtual network topology has two bridges; without that, it might go even higher).
- I looked into libvirt’s PCIe configuration a bit. To make the XML cleaner, I put all devices on bus 0, so you don’t see `pcie-root-port` or `pcie-pci-bridge` devices in the XML. In my tests this didn’t lead to any noticeable performance loss; I’m not sure how much the PCIe topology actually matters.

## Configuring TCRP

Boot into the TCRP Loader, log in via SSH with username `tc` and default password `P@ssw0rd`, then run:

```bash
./rploader.sh fullupgrade
./rploader.sh serialgen DS918+ realmac
./rploader.sh identifyusb
./rploader.sh satamap
cat user_config.json | jq '.extra_cmdline.SataPortMap = "12"' | jq '.extra_cmdline.DiskIdxMap = "1000"' | tee user_config.json
./rploader.sh ext ds918p-7.1.1-42962 add https://github.com/pocopico/redpill-load/raw/develop/redpill-virtio/rpext-index.json
./rploader.sh ext ds918p-7.1.1-42962 add https://github.com/pocopico/redpill-load/raw/develop/redpill-acpid/rpext-index.json
./rploader.sh build ds918p-7.1.1-42962
```

Reboot, use Synology Assistant to find the DSM instance, upload the 7.1.1-42962 PAT file, and you’re done.

It looks simple, but this is the result of a dozen or so trial-and-error runs. There are a few gotchas:

- Regarding the `SataPortMap` and `DiskIdxMap` parameters: given the hardware layout, I have two controllers, one Q35 SATA controller and one SCSI controller driven by `virtio-scsi`. The Q35 controller is unused, so its `DiskIdxMap` is set to `10`, which exceeds the number of disks and will be ignored. The SCSI controller has two disks attached, so its `SataPortMap` is 2, and its `DiskIdxMap` is `00`, meaning from the first disk onward.
- The `acpid` extension is added so that pressing the power button can trigger a proper shutdown, allowing clean shutdown via `virsh shutdown DSM`.
- ~~About the VirtIO drivers: when I installed, the VirtIO driver in the TCRP repo didn’t yet include `7.1.1-42962`, only up to `7.1.0-42661`. So I cloned the repo, added `ds918p_42962`, and manually added the extension. It turned out the driver itself worked fine and only needed some configuration; this should be resolved once the TCRP repo is updated. Initially I installed version `7.1.0-42661`, which worked, and then I tried updating DSM directly from within the system, but the update failed and the system wouldn’t boot afterwards.~~ The repo has since been updated and can be used directly.
- ~~One odd point was that `serialgen` takes a model name as a parameter, while `build` takes a CPU platform code. During my experiments I tried several models (DS918+, DS3622xs+, DS3617xs), which made this especially annoying. I finally found a good mapping table in reference #3 and am including it here:~~ In the latest version the commands have been unified, but I’ll leave the table here for reference.

    |DSM Platform|DS918+|DS3622xs+|DS920+|DS1621+|DS3617xs|DVA3221|DS3615xs|
    |---|---|---|---|---|---|---|---|
    |Architecture|apollolake|broadwellnk|geminilake|v1000|broadwell|denverton|bromolow|
    |Drive Slot Mapping|sataportmap<br/>/diskidxmap|sataportmap<br/>/diskidxmap|device tree|device tree|sataportmap<br/>/diskidxmap|sataportmap<br/>/diskidxmap|sataportmap<br/>/diskidxmap|
    |QuickSync Transcoding|Yes|No|Yes|No|No|No|No|
    |NVMe Cache Support|Yes|Yes|Yes|Yes|Yes (as of 7.0)|Yes|No|
    |RAIDF1 Support|No|Yes|No|No|Yes|No|Yes|
    |Oldest CPU Supported|Haswell|any x86-64|Haswell|any x86-64|any x86-64|Haswell|any x86-64|
    |Max CPU Threads|8|24|8|16|24 (as of 7.0)|16|16|
    |Key Note|currently best for most users|best for very large installs||AMD Ryzen|obsolete use DS3622xs+|AI/Deep Learning nVIDIA GPU|obsolete use DS3622xs+|
---
title: "libvirt安装黑群晖"
date: 2022-10-21
slug: DSM-on-libvirt
tags:
- DSM
---

最近升级NAS之后，空余出来2块10T的硬盘组不进阵列，原本想拿来多做一份ZFS的备份，结果发现zrepl不支持多目的地，就不想折腾了。正好又折腾好了WebVirtCloud，
就想试试看搞个黑群晖，将两块硬盘直通进去，这样又能多一个折腾的玩具。

<!--more-->

说起来自从裸机安装Ubuntu做NAS以来，别的功能都还好，就是虚拟机的管理界面我一直不满意。之前用了个很别扭的[方案](https://github.com/m-bers/docker-virt-manager)，原理是用Broadway将桌面版的
Virtual Machine Manager转成Web版，用是能用，但是虚拟机的画面奇卡无比。WebVirtCloud使用Python写的纯Web版，虽然跟Proxmox，Unraid这些成熟产品不能比，但效果还是好不少，凑合着用了。

言归正传，先上各种参考资料：

1. [QEMU系列-黑群晖DSM7.0.1](https://www.wnark.com/archives/174.html)
1. [Tutorial: Install/Migrate to DSM 7.x with TinyCore RedPill (TCRP) Loader](https://xpenology.com/forum/topic/62221-tutorial-installmigrate-to-dsm-7x-with-tinycore-redpill-tcrp-loader/)
1. [DSM 7.x Loaders and Platforms](https://xpenology.com/forum/topic/61634-dsm-7x-loaders-and-platforms/)
1. [Tutorial: Install DSM 7.1 on UNRAID 6.10.3](https://xpenology.com/forum/topic/63333-tutorial-install-dsm-71-on-unraid-6103)
1. [群晖DSM7.X版本自动获取SataPortMap和DiskIdxMap的教程](https://wp.gxnas.com/11876.html)

## 环境

宿主机系统是Ubuntu 22.04，但是由于libvirt是安装在Docker容器里的，容器镜像基于Ubuntu 20.04 （因为22.04的libvirt会因为CGroup2的原因启动不了虚拟机），因此QEMU版本是4.2.1，libvirt版本6.0.0。

黑群晖的型号是DS918+，版本是截至当前最新的7.1.1-42962。

目前DSM7的引导基本都是用TinyCore RedPill (TCRP)做的，TCRP版本是~~0.9.2.8~~ 0.9.2.9.

## 硬件配置

WebVirtCloud在硬件配置方面只能满足最基本的需求，好在支持XML配置。基本上基于KVM的虚拟机方案都会用到libvirt的XML配置，所以我这里直接给出我折腾成功的配置。

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

最终运行的QEMU命令如下：

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

简单说明一下这个配置：

- 芯片组选的Q35，BIOS用UEFI，因此TCRP的镜像也要下载UEFI的版本
- CPU选`host-passthrough`。DS918+只支持Haswell及以后的CPU，具体的看参考链接
- USB总线model选`qemu-xhci`，这个是USB3.0，默认的是USB2.0
- 两块直通的硬盘挂在了SCSI总线上。SATA总线也可以，但是QEMU虚拟出来的这个Q35的SATA控制器，只支持SATA 1.0的速度，也就是1.5Gbps，实测下来读取速度最高150MiB/s。况且TCRP还把这个标记为了Bogus Q35，所以最好别用
- SCSI控制器用的是`virtio-scsi`，这个需要额外安装驱动，不过VirtIO性能好，可以最大程度发挥硬盘性能
- 网卡用的是`virtio`，需要安装驱动。我先后试过`e1000e`和`vmxnet3`。`e1000e`用了之后DS918+会掉IP，无法引导。`vmxnet3`在VMWare里是个万兆网卡，结果我试下来在QEMU里链路速率只有千兆，用iperf3做多线程测试的时候，
    甚至只能跑到600Mbps。VirtIO性能好，iperf3与宿主机测试可以跑到26Gbps（我的虚拟网络拓扑有两个bridge，不然说不定能跑到更高）
- 我研究了下libvirt的PCIe配置，为了XML看起来更简洁些，我把所有设备都挂到了bus 0上，所以在XML里没有`pcie-root-port`和`pcie-pci-bridge`设备，目前测试下来感觉并没有性能损失，不清楚PCIe设备的拓扑图到底会有啥影响

## 配置TCRP

开机引导至TCRP Loader，用SSH登陆，用户名是`tc`，密码默认是`P@ssw0rd`，然后依次运行：

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

重启，用Synology Assitant找到DSM，上传7.1.1-42962的PAT文件，搞定收工。

看着挺简单的，但是这是我来来回回折腾了十几遍得出的结果，其中有几个坑：

- 关于`SataPortMap`和`DiskIdxMap`这两个参数的配置，根据硬件配置，我有两个控制器，一个是Q35的SATA控制器，一个是`virtio-scsi`驱动的SCSI控制器，Q35的控制器没有使用，所以对应的`DiskIdxMap`设为了`10`，
    超过了硬盘数量上限，会被忽略，SCSI控制器上挂了两块硬盘，所以对应的`SataPortMap`是2，对应的`DiskIdxMap`是`00`，表示从第一块盘开始
- 增加的extention `acpid`是为了支持按关机键可以关机，这样可以通过`virsh shutdown DSM`正常关机
- ~~VirtIO驱动的问题，我安装的时候TCRP仓库里的VirtIO驱动还没加上`7.1.1-42962`，只更新到`7.1.0-42661`，于是我克隆了仓库，加入了`ds918p_42962`，然后手动添加extension。事实证明驱动本身可以使用，
    只需要加个配置就行，这个问题等过一段时间TCRP仓库更新了就能解决。一开始的时候我安装了`7.1.0-42661`版本，也能成功，然后尝试在DSM系统里直接更新，结果失败，重启后就进不了系统了~~ 仓库已更新，可直接使用
- ~~比较奇怪的一点是`serialgen`命令接收的是型号参数，而`build`命令接收的是CPU代号，因为我折腾的过程中试过好几个型号（DS918+，DS3622xs+，DS3617xs），所以特别烦这个，最后在参考资料#3中找到了好用的对应表，
    我把那个贴这里供参考：~~ 最新版居然统一了命令，表格就留在这里作参考了

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
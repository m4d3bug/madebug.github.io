---
title: Hello Kickstart + PXE + UEFI
date: 2019-09-14 12:21:48
categories:
- "Ops "
tags:
- "Kickstart "
- "DHCP "
- "TFTP "
- "PXE "
- "Httpd "
- "RHEL 7 "
- "Linux "
- "UEFI "
- "Ancanda "
---

*In this post, I am going to markdown how I tested unattended kickstart installation for UEFI in vm15 by using PXE.*

*Will increase uefi support based on [previous environment](https://blog.madebug.net/Ops/2019-08-18-hello-kickstart-pxe-mbr.html).*

## *Environmental preparation*

### *Unpacking packages*

``` nohighlight
[root@localhost ~]# cp -pr /mnt/RHEL-7/7.4/Packages/shim-x64-12-1.el7.x86_64.rpm /tmp
[root@localhost tftpboot]# cp -pr /mnt/RHEL-7/7.4/Packages/grub2-efi-x64-2.02-0.64.el7.x86_64.rpm /tmp
[root@localhost ~]# cd /var/lib/tftpboot/
[root@localhost tftpboot]# rpm2cpio /tmp/shim-x64-12-1.el7.x86_64.rpm |cpio -dimv
./boot/efi/EFI/BOOT/BOOTX64.EFI
./boot/efi/EFI/BOOT/fbx64.efi
./boot/efi/EFI/redhat/BOOT.CSV
./boot/efi/EFI/redhat/BOOTX64.CSV
./boot/efi/EFI/redhat/mmx64.efi
./boot/efi/EFI/redhat/shim.efi
./boot/efi/EFI/redhat/shimx64-redhat.efi
./boot/efi/EFI/redhat/shimx64.efi
12773 blocks
[root@localhost tftpboot]# rpm2cpio /tmp/grub2-efi-x64-2.02-0.64.el7.x86_64.rpm |cpio -dimv
./boot/efi/EFI/redhat
./boot/efi/EFI/redhat/fonts
./boot/efi/EFI/redhat/fonts/unicode.pf2
./boot/efi/EFI/redhat/grubx64.efi
./boot/grub2/grubenv
./etc/grub2-efi.cfg
7054 blocks
[root@localhost tftpboot]# tree -L 2 .
.
├── boot
│   ├── efi
│   └── grub2
├── etc
│   └── grub2-efi.cfg -> ../boot/efi/EFI/redhat/grub.cfg
├── pxelinux
│   ├── initrd.img
│   ├── pxelinux.0
│   ├── pxelinux.cfg
│   ├── vesamenu.c32
│   └── vmlinuz
└── usr
    ├── bin
    └── share

9 directories, 5 files
```

## *Configure PXE Server*

### *Install required packages and copy boot menu program file*

```nohighlight
[root@localhost ~]# mkdir /var/lib/tftpboot/uefi
[root@localhost tftpboot]# cp boot/efi/EFI/redhat/shim.efi /var/lib/tftpboot/uefi/
[root@localhost tftpboot]# cp boot/efi/EFI/redhat/grubx64.efi /var/lib/tftpboot/uefi/

[root@localhost ~]# cp -pr /mnt/RHEL-7/7.4/Packages/shim-x64-12-1.el7.x86_64.rpm /tmp
[root@localhost ~]# cp -pr /mnt/RHEL-7/7.4/Packages/grub2-2.02-0.64.el7.x86_64.rpm /tmp
[root@localhost ~]# cd /var/lib/tftpboot/
[root@localhost tftpboot]# rpm2cpio /tmp/syslinux-4.05-13.el7.x86_64.rpm |cpio -dimv
[root@localhost tftpboot]# ls
pxelinux  usr
[root@localhost tftpboot]# more /var/lib/tftpboot/uefi/grub.cfg
set timeout=60
  menuentry 'RHEL 7' {
  linuxefi uefi/vmlinuz ip=dhcp inst.repo=http://192.168.188.174/RHEL-7/7.4/Server/x86_64 inst.gpt
  initrdefi uefi/initrd.img
}
[root@localhost tftpboot]# cp /var/lib/tftpboot/pxelinux/{vmlinuz,initrd.img} /var/lib/tftpboot/uefi/
[root@localhost ~]# cp /var/lib/tftpboot/usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/pxelinux/
[root@localhost ~]# cp /var/lib/tftpboot/usr/share/syslinux/vesamenu.c32 /var/lib/tftpboot/pxelinux/
[root@localhost tftpboot]# umount /mnt/
[root@localhost tftpboot]# tree -L 2 .
.
├── pxelinux
│   ├── initrd.img
│   ├── pxelinux.0        ------> Bootloader for Bios.
│   ├── pxelinux.cfg      ------> Default dir of pxeliunx.0's config.
│   ├── vesamenu.c32      ----- > Use for draw PXE menu.
│   └── vmlinuz           ----- > Kernel
└── usr
    ├── bin
    └── share

5 directories, 4 files
```

### *Boot up should get this:*

#### *Server*

![dhcplog.png](https://i.loli.net/2019/09/14/r62YRjlST4QObtg.png)

#### *Client*

![dhcpok.png](https://i.loli.net/2019/09/14/aPyrBv4pixcd6mL.png)

![UEFIboot.png](https://i.loli.net/2019/09/14/1O5BijJv9dlQpTa.png)

## *Configure Network Install*

### *Monitor httpd*

``` nohighlight
[root@localhost ~]# tcpdump -i ens33 port 80 and host 172.16.1.100 -vvv
```

### *Create a  vm machines in the same LAN without iso.*

![](https://i.loli.net/2019/08/19/w8lkvhm1aFeg5ur.jpg)

![vmUEFI.png](https://i.loli.net/2019/09/14/iDueByIX3vYJWT1.png)

*Enable network boot on BIOS settings of client computer and start it, then installation menu is shown, push Enter key to proceed to install.*

![ancanda.png](https://i.loli.net/2019/09/14/5iecmlxNtYS3jyk.png)

## *Configure Kickstart Install*

```nohighlight
[root@localhost ~]# mkdir /var/www/html/ks 
[root@localhost ~]# system-config-kickstart
```

### *Custom your ks.cfg*

*Also can generate from [here](https://access.redhat.com/labs/kickstartconfig/).*

![](https://i.loli.net/2019/08/18/NiJG2l7BSgHyvWe.png)

![](https://i.loli.net/2019/10/08/BzDQKP1vZGds5W3.png)

![](https://i.loli.net/2019/08/18/K1WNP4OGlFILhCS.png)

![](https://i.loli.net/2019/08/18/GFX76hIsiwB9AzO.png)

![](https://i.loli.net/2019/08/18/VcaJwYrvNjkhDOF.png)

![](https://i.loli.net/2019/08/20/YzJ91PCfjvQFKs2.jpg)

![](https://i.loli.net/2019/08/18/8y5D3huAg2siJ6L.png)

![uefi-ks.png](https://i.loli.net/2019/09/14/HQ2bSTAxwv98tsl.png)

### *Configure packages I need*

```nohighlight
[root@localhost ~]# cat ~/anaconda-ks.cfg |grep -A 19 %packages >> /var/www/html/ks/uefi-ks.cfg
%packages
@^graphical-server-environment
@base
@core
@desktop-debugging
@dial-up
@fonts
@gnome-desktop
@guest-agents
@guest-desktop-agents
@hardware-monitoring
@input-methods
@internet-browser
@multimedia
@print-client
@x11
chrony
kexec-tools

%end
```

### *Configure disk*

``` nohighlight
[root@localhost ~]# sed -i '/bootloader --location=mbr/a autopart --type=lvm' /var/www/html/ks/uefi-ks.cfg
[root@localhost ~]# cat /var/www/html/ks/uefi-ks.cfg
#version=DEVEL
# System authorization information
auth --enableshadow --passalgo=sha512
# Use network installation
url --url="http://192.168.188.174/RHEL-7/7.4/Server/x86_64/"
#repo --name="Server-ResilientStorage" --baseurl=http://10.66.192.123/rhel7u5//addons/ResilientStorage
# Use graphical install
#graphical
text
# Run the Setup Agent on first boot
firstboot --enable
ignoredisk --only-use=sda
# Keyboard layouts
keyboard --vckeymap=us --xlayouts='us'
# System language
lang en_US.UTF-8
# Network information
#network  --bootproto=dhcp --device=eth0 --ipv6=auto --activate
network --onboot yes --device eth0 --bootproto dhcp --noipv6
network  --hostname=localhost.localdomain
# Root password
rootpw --iscrypted $1$n5NVIcJj$honUQjAMpqWcJcTk5MdPq/
# System services
services --enabled="chronyd"
# System timezone
timezone Asia/Shanghai --isUtc
# System bootloader configuration
bootloader --append=" crashkernel=128M" --location=mbr --boot-drive=sda
# Partition clearing information
#zerombr
clearpart --all --initlabel
# Disk partitioning information
#%pre --log=/root/pre_log

#/usr/bin/dd bs=512 count=10 if=/dev/zero of=/dev/sda
#/usr/sbin/parted --script /dev/sda mklabel gpt
#/usr/sbin/parted --script /dev/sda print 
#/usr/bin/sleep  30

#%end

part pv.146 --fstype="lvmpv" --ondisk=sda --size=10000
part /boot/efi --fstype=efi --size=1024 --ondisk=sda
part /boot --fstype="xfs" --ondisk=sda --size=1024
volgroup rhel00 pv.146
logvol /  --fstype="xfs" --name=root --vgname=rhel00 --percent=100
#logvol /var/log  --fstype="xfs" --grow --name=log --vgname=rhel00 --percent=50
#logvol swap  --fstype="swap" --size=2047 --name=swap --vgname=rhel00

%packages
@^graphical-server-environment
@base
@^minimal
@core
@desktop-debugging
@dial-up
@fonts
@gnome-desktop
@guest-agents
@guest-desktop-agents
@hardware-monitoring
@input-methods
@internet-browser
@multimedia
@print-client
@x11
chrony
kexec-tools
%end
%addon com_redhat_kdump --enable --reserve-mb='128M'
%end
%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end


```

### *Edit the boot menu*

```nohighlight
[root@localhost ~]# more /var/lib/tftpboot/uefi/grub.cfg  
set timeout=3
  menuentry 'RHEL 7' {
  linuxefi uefi/vmlinuz ip=dhcp inst.repo=http://192.168.188.174/RHEL-7/7.4/Server/x86_64 inst.ks=http://192.168.188.174/ks/uefi-ks.cfg inst.gpt
  initrdefi uefi/initrd.img
}
```

*Start the vm machine which is enabled network booting on BIOS settings, then PXE boot menu is shown. After 10 seconds later, installation process starts and will finish and reboot automatically.*

## *Done*

*Simply tried to complete the unattended installation of RHEL7 by using Kickstart with PXE.*

## *Acknowledgements*

- https://www.cnblogs.com/boowii/p/6475921.html
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/index
- https://www.server-world.info/en/note?os=CentOS_7&p=pxe&f=1


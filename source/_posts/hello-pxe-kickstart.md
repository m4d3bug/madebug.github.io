---
title: Hello PXE With Kickstart
date: 2019-08-18 12:21:48
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
- "BIOS "
- "UEFI "
- "Anaconda "
---

*In this post, I am going to markdown how to build a unattended PXE Server by using kickstart in BIOS and UEFI.*

*And we will run these services on the same machine with PXE: TFTP, DHCP & httpd.*

## *Environmental preparation*

### *Turn off the selinux ,firewalld and iptables*

``` nohighlight
~]# cat /etc/selinux/config |grep ^SELINUX=
SELINUX=disabled
~]# systemctl stop firewalld.service
~]# systemctl disable firewalld.service
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
~]# iptables -F
```

### *Stop Virtual Network Editor's DHCP services*

![](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/20191012215006.png)

### *Configure custom ip*

![](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/20191012215142.png)

## *Configure PXE Server*

### *Install required packages and copy boot menu program file*

```nohighlight
~]# yum -y install syslinux tftp-server dhcp
~]# mkdir /var/lib/tftpboot/pxelinux
~]# mkdir -p /mnt/RHEL-7/7.4
~]# mount /dev/sr0 /mnt/RHEL-7/7.4
mount: /dev/sr0 is write-protected, mounting read-only
~]# cp -pr /mnt/RHEL-7/7.4/Packages/syslinux-4.05-13.el7.x86_64.rpm /tmp
~]# cd /var/lib/tftpboot/
tftpboot]# rpm2cpio /tmp/syslinux-4.05-13.el7.x86_64.rpm |cpio -dimv
tftpboot]# ls
pxelinux  usr
```

### *Preparation for BIOS*

```nohighlight
~]# mkdir /var/lib/tftpboot/pxelinux/pxelinux.cfg
~]# cat >> /var/lib/tftpboot/pxelinux/pxelinux.cfg/default << EOF
default vesamenu.c32
prompt 1
timeout 100

label linux
  menu label ^Install system
  menu default
  kernel vmlinuz
  append initrd=initrd.img ip=dhcp inst.repo=http://192.168.188.174/RHEL-7/7.4/Server/x86_64/ inst.ks=http://192.168.188.174/ks/bios-ks.cfg
label vesa
  menu label Install system with ^basic video driver
  kernel vmlinuz
  append initrd=initrd.img ip=dhcp inst.xdriver=vesa nomodeset inst.repo=http://192.168.188.174/RHEL-7/7.x/Server/x86_64/os/
label rescue
  menu label ^Rescue installed system
  kernel vmlinuz
  append initrd=initrd.img rescue
label local
  menu label Boot from ^local drive
  localboot 0xffff
EOF
~]# cp /mnt/RHEL-7/7.4/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/pxelinux/
~]# cp /var/lib/tftpboot/usr/share/syslinux/{pxelinux.0,vesamenu.c32}  /var/lib/tftpboot/pxelinux/
```

### *Preparation for UEFI*

*You cannot use pxelinux to boot UEFI and replace it with grubx64.efi*.

```nohighlight
~]# cp /mnt/RHEL-7/7.4/EFI/BOOT/grubx64.efi /var/lib/tftpboot/
~]# cat >> /var/lib/tftpboot/grub.cfg << EOF
set timeout=9
menuentry 'Install Red Hat Enterprise Linux 7.4' {
        linuxefi pxelinux/vmlinuz ip=dhcp inst.repo=http://192.168.188.174/RHEL-7/7.4/Server/x86_64 inst.ks=http://192.168.188.174/ks/uefi-ks.cfg inst.gpt
        initrdefi pxelinux/initrd.img
}
EOF
```

## *Start TFTP server*

``` nohighlight
~]# cat /etc/xinetd.d/tftp |grep disable
	disable			= no
~]# systemctl start tftp
~]# systemctl enable tftp
Created symlink from /etc/systemd/system/sockets.target.wants/tftp.socket to /usr/lib/systemd/system/tftp.socket.
```

## *Start DHCP server*

```nohighlight
~]# cat >> /etc/dhcp/dhcpd.conf << EOF
# FILL THIS UP
#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp*/dhcpd.conf.example
#   see dhcpd.conf(5) man page
#
option space pxelinux;
option pxelinux.magic code 208 = string;
option pxelinux.configfile code 209 = text;
option pxelinux.pathprefix code 210 = text;
option pxelinux.reboottime code 211 = unsigned integer 32;
option architecture-type code 93 = unsigned integer 16;

subnet 192.168.188.0 netmask 255.255.255.0 {
  option routers 192.168.188.2;
  range 192.168.188.200 192.168.188.253;

  class "pxeclients" {
      match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
      next-server 192.168.188.174;

      if option architecture-type = 00:07 {
        filename "grubx64.efi";
      } else {
        filename "pxelinux/pxelinux.0";
      }
  }
}
EOF
~]# systemctl start dhcpd 
~]# systemctl enable dhcpd 
Created symlink from /etc/systemd/system/multi-user.target.wants/dhcpd.service to /usr/lib/systemd/system/dhcpd.service.
~]# rm -rf /var/lib/tftpboot/usr
~]# tree -L 3 /var/lib/tftpboot/
/var/lib/tftpboot/
├── grub.cfg                     <--- Config file for UEFI
├── grubx64.efi                  <--- bootloader program for UEFI
└── pxelinux
    ├── initrd.img               <--- images for kernel in bootup progress
    ├── pxelinux.0               <--- bootloader program for BIOS
    ├── pxelinux.cfg             <--- default dir to store the config for pxelinux.0
    │   └── default              <--- Config file for BIOS
    ├── vesamenu.c32             <--- header file for default
    └── vmlinuz                  <--- kernel in boot progress

2 directories, 7 files

```

### *Check if DHCP works as expected:*

*Try to start the machine on the same LAN for a test, you can view its DHCP request in the log.*

![](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/20191012215745.png)

![](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/20191012215234.png)

## *Configure Network Installation source*

### *Use httpd to provide repo source*

``` nohighlight
~]# yum -y install httpd
~]# rm -f /etc/httpd/conf.d/welcome.conf
~]# cat >> /etc/httpd/conf.d/iso.conf << EOF
# Create this
Alias /RHEL-7/7.4/Server/x86_64 /mnt/RHEL-7/7.4
<Directory /mnt/RHEL-7/7.4>
    Options Indexes FollowSymLinks
    Require ip 0.0.0.0
</Directory>
EOF
~]# systemctl enable httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
~]# systemctl start httpd 
~]# curl -v http://192.168.188.174/RHEL-7/7.4/Server/x86_64/
* About to connect() to 192.168.188.174 port 80 (#0)
*   Trying 192.168.188.174...
* Connected to 192.168.188.174 (192.168.188.174) port 80 (#0)
> GET /RHEL-7/7.4/Server/x86_64/ HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 192.168.188.174
> Accept: */*
>
< HTTP/1.1 200 OK               <================ MAKE SURE WORKING.
< Date: Thu, 02 Apr 2020 07:29:58 GMT
< Server: Apache/2.4.6 (Red Hat Enterprise Linux)
< Content-Length: 3294
< Content-Type: text/html;charset=ISO-8859-1
...
* Connection #0 to host 192.168.188.174 left intact

# FOR TROUBLESHOOTING.
~]# journalctl -xefu httpd
~]# tcpdump -i ens33 port 80 and host 192.168.188.174 -vvv >> tcpdump.out
```

### *Check if repo source works as expected:*

*Create a vm machines in the same LAN without iso.*

![](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/20191012215904.png)

![](https://raw.githubusercontent.com/m4d3bug/images-of-website/master/blog/20191012215957.png)

## *Configure Kickstart Config*

### *Create your ks dir*

```nohighlight
~]# mkdir /var/www/html/ks
```

### *Custom your ks.cfg for BIOS*

```nohighlight

~]# cat >> /var/www/html/ks/bios-ks.cfg << EOF
#platform=x86, AMD64, or Intel EM64T
#version=DEVEL
# Install OS instead of upgrade
install
# Keyboard layouts:
keyboard 'us'
# Root password
rootpw --iscrypted $1$MHXtAAGd$k9MrCZ5lII6h9cqL1w15h/
# System language
lang en_US
# System authorization information
auth  --useshadow  --passalgo=sha512
# Use graphical install
graphical
firstboot --disable
# SELinux configuration
selinux --disabled


# Firewall configuration
firewall --disabled
# Network information
network  --bootproto=dhcp --device=ens33
# Reboot after installation
reboot
# System timezone
timezone Africa/Abidjan
# Use network installation
url --url="http://172.16.238.1/RHEL-7/7.7/Server/x86_64"
# System bootloader configuration
bootloader --location=mbr --boot-drive=sda
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --all --initlabel
# Disk partitioning information
part /boot --fstype=xfs --size=500
part pv.009009 --grow --size=1
volgroup VolGroup --pesize=4096 pv.009009
logvol / --fstype=xfs --name=lv_root --vgname=VolGroup --percent=100
logvol swap --fstype=swap --name=lv_swap --vgname=VolGroup --recommended

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

%addon com_redhat_kdump --enable --reserve-mb='auto'
%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end
EOF
```

### *Custom your ks.cfg for UEFI*

```nohighlight
~]# cat >> /var/www/html/ks/uefi-ks.cfg << EOF
#platform=x86, AMD64, or Intel EM64T
#version=DEVEL
# Install OS instead of upgrade
install
# Keyboard layouts
keyboard 'us'
# Root password
rootpw --iscrypted $1$CnXC/qBC$W8vXGWY/oN7RvtVGh0mQT1
# System language
lang en_US
# System authorization information
auth  --useshadow  --passalgo=sha512
# Use graphical install
graphical
firstboot --disable
# SELinux configuration
selinux --disabled


# Firewall configuration
firewall --disabled
# Network information
network  --bootproto=dhcp --device=ens33
# Reboot after installation
reboot
# System timezone
timezone Africa/Abidjan
# Use network installation
url --url="http://172.16.238.1/RHEL-7/7.7/Server/x86_64"
# System bootloader configuration
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
# Clear the Master Boot Record
# zerombr
# Partition clearing information
clearpart --all --initlabel
# Disk partitioning information
part /boot/efi --fstype=efi --size=200 --asprimary
part /boot --fstype=xfs --size=500
part pv.009009 --grow --size=1
volgroup VolGroup --pesize=4096 pv.009009
logvol / --fstype=xfs --name=lv_root --vgname=VolGroup --percent=100
logvol swap --name=lv_swap --vgname=VolGroup --recommended

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

%addon com_redhat_kdump --enable --reserve-mb='128M'
%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end
EOF
```

## *Done*

*Simply tried to complete the unattended installation of RHEL7 by using PXE with Kickstart.*

## *Acknowledgements*

- https://www.cnblogs.com/boowii/p/6475921.html
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/installation_guide/index#chap-installation-server-setup
- https://www.server-world.info/en/note?os=CentOS_7&p=pxe&f=1


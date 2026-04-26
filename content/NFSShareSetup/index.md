---
title: "NFS Share Setup CentOS"
subtitle: "General Guide for CentOS 8"
date: 2020-02-28T12:44:37-04:00
lastmod: 2020-05-29T12:44:37-04:00
draft: false
author: "Nathan Higley"
authorLink: ""
description: ""

tags: [NFS, server, CentOS]
categories: [Guides, CentOS]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---
# CentOS NFS Share Setup

## Misc Things

Get temporary network:
>sudo dhclient

Reboot faster:
>sudo init 6

## Format /dev/sdb

Install [rpmfusion](https://rpmfusion.org/Configuration) for exfat support.

>sudo dnf install --nogpgcheck https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo dnf install --nogpgcheck https://download1.rpmfusion.org/free/el/rpmfusion-free-release-8.noarch.rpm https://download1.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-8.noarch.rpm
sudo dnf config-manager --enable PowerTools
sudo dnf update
sudo dnf install exfat-utils fuse-exfat


Format drive:

> sudo mkfs.exfat /dev/sdb



Mount:


>sudo mkdir /mount-point

>sudo mount /dev/sdb /mount-point

Setup fstab entry:
>sudo vim /etc/fstab

```
/dev/sdb    /mount-point  exfat rw,async,umask=0 0 0
```


## Setup NFS

Tutorial used:
[Setting Up an NFS Server and Client on CentOS 7.2](https://www.howtoforge.com/tutorial/setting-up-an-nfs-server-and-client-on-centos-7/)

1) Install software

>yum install nfs-utils

2) Setup Startup Scripts

>systemctl enable nfs-server.service

>systemctl start nfs-server.service

3) Setup NFS
Make the config
>sudo vim /etc/export:
```
/mount-point 192.168.0.2(rw, no_squash_root) 192.168.0.3(rw, no_squash_root)
```

Export the config:
>sudo exportfs -a

Those IP's should now have read/write access to the share.

Update SELinux Boolean:
>sudo setsebool -P nfs_export_all_rw 1


4) Setup final networking

Setup Static IP on CentOS:

Edit Networking config: 
>sudo vim /etc/sysconfig/network-scripts/ifcfg-\<interfacename>
```
NETMASK=225.255.255.0
GATEWAY=192.168.0.1
IPADDR=192.168.0.2
ONBOOT=yes
BOOTPROTO=none
USERCTL=no
```

Allow through firewall:

Use this command to find ports
>rpcinfo -p

Then enable the ports:
>firewall-cmd --permanent --add-port=\<portnumber>/\<tcp or udp>

>firewall-cmd --reload


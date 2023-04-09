---
title: "Arch Linux Full Encryption Installation Guide"
subtitle: "With LVM and LUKS"
date: 2020-04-06T12:52:52-04:00
lastmod: 2020-05-29T12:52:52-04:00
draft: false
author: "Nathan Higley"
authorLink: ""
description: ""

tags: [Arch Linux, Linux, Encryption]
categories: [Security, Linux]

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

# Arch Linux w/ Fully Encrypted Filesystem

This guide will show step by step how to create a clean Arch Linux install with a fully encrypted filesystem.  This means that even the boot partition will be encrypted.  The only unencrypted partition on the disk will be the EFI partition which could be configured later to use secure boot.

Assuming an EFI system with GPT disk.

### Basic Install Stuff

Make sure you can hit the outside world:
>ping google.com

If not run dhcpcd:
>dhcpcd

Set the time:
>timedatectl set-ntp true

### Setup the Disk (where the magic happens)

#### Create Partitions
Assuming /dev/sda is the device you want to install to.

>parted /dev/sda

>mklabel gpt

>mkpart ESP fat32 1MiB 200MiB

>set 1 boot on

>name 1 efi

>mkpart primary 800MiB 100%

>set 2 lvm on

>name 2 lvm

>print

#### Setup LUKS

Encrypt the partition
>cryptsetup luksFormat --type luks1 /dev/sda2

Open the partition
>cryptsetup open /dev/sda2 encrypted-lvm

#### Setup LVM
Create a Physical Volume
>pvcreate /dev/mapper/encrypted-lvm

Create a volume group
>vgcreate arch /dev/mapper/encrypted-lvm

Create logical volumes in group
>lvcreate -n home -L \<SIZE>G arch

>lvcreate -n root -L \<SIZE>G arch

>lvcreate -n boot -L 600M arch

>lvcreate -n swap -L \<SIZE> -C y arch

#### Format Partitions
Format the EFI partition
>mkfs.fat -F32 /dev/sda1

Format the Volume Groups
>mkfs.ext4 -L boot /dev/mapper/arch-boot

>mkfs.btrfs -L root /dev/mapper/arch-root

>mkfs.btrfs -L home /dev/mapper/arch-home

>mkswap /dev/mapper/arch-swap

### Mount Partitions

Configure Swap
> swapon /dev/mapper/arch-swap
> swapon -a; swapon -s
>
Mount the filesystem
>mount /dev/mapper/arch-root /mnt

>mkdir -p /mnt/{home,boot}

>mount /dev/mapper/boot /mnt/boot

>mount /dev/mapper/arch-home /mnt/home

>mkdir /mnt/boot/efi

>mount /dev/sda1 /mnt/boot/efi

Check how everything is mounted
>lsblk -f

### Install the Base System
Install base packages
>pacstrap /mnt base efibootmgr btrfs-progs grub linux-zen linux-firmware lvm2

Generate Fstab
>genfstab -U -p /mnt > /mnt/etc/fstab

Chroot into new install
>arch-chroot /mnt /bin/bash

### Configure the Install

#### Normal Things
Set the time zone
>ln -sf /usr/share/zoneinfo/Region/City /etc/localtime

>hwclock --systohc

Localization
Uncomment "en_US.UTF-8 UTF-8" from /etc/locale.gen
>locale-gen

>echo "LANG=en_US.UTF-8" > /etc/locale.conf

Set the hostname
>echo "hostname" > /etc/hostname

Edit /etc/hosts to
```
127.0.0.1	localhost
::1	        localhost
127.0.1.1	hostname.localdomain	hostname
```

#### Encryption Unlock

Generate a keyfile for the root filesystem:
>dd bs=512 count=4 if=/dev/random of=/root/encrypted-lvm.keyfile iflag=fullblock

>chmod 000 /root/encrypt-lvm.keyfile

>cryptsetup -v luksAddKey /dev/sda2 /root/encrypted-lvm.keyfile

Edit /etc/mkinitcpio.conf and add keyboard keymap encrypt lvm2 to hooks and add keyfile to files
```
FILES=(/root/encrypted-lvm.keyfile)
HOOKS=(base udev autodetect keyboard keymap consolefont modconf block encrypt lvm2 filesystems fsck)
```

Then generate the initramfs image:
>mkinitcpio -p linux-zen

Secure the embedded keyfile:
>chmod 600 /boot/initramfs-linux*

Configure grub, edit /etc/default/grub:
```
GRUB_ENABLE_CRYPTODISK=y
GRUB_CMDLINE_LINUX="... cryptdevice=/dev/sda2:encrypted-lvm ... cryptkey=rootfs:/root/encrypted-lvm.keyfile"
```

Install grub:
>grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB --recheck

>grub-mkconfig -o /boot/grub/grub.cfg

#### Final Cleanup
Set a root password
>passwd
Add a normal user account if desired.

### Reboot into New Install

>exit

>umount -R /mnt

>reboot

The system should be configured with full disk encryption and an encrypted boot partition.



### References
1) [Installation guide - ArchWiki](https://wiki.archlinux.org/index.php/Installation_guide)
2) [dm-crypt/Encrypting an entire system - ArchWiki](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Encrypted_boot_partition_(GRUB))
3) [Install Arch Linux with full hard drive encryption using luks encryption \| ComputingForGeeks](https://computingforgeeks.com/install-arch-linux-luks-encryption/)


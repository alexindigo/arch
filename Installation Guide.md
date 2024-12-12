# Installation Guide

- [Key Points](#key-points)
- [Assumptions](#assumptions)
- [Prerequisites](#prerequisites)
- [0. Prepare installer](#0-prepare-installer)
- [1. Prepare install environment](#1-prepare-install-environment)
- [2. Partition disk](#2-partition-disk)
- [References](#references)

## Key Points

- BTRFS
- Encryption (LUKS) + encrypted swap
- Snapshots, auto-update GRUB
- Hibernation
- 
## Assumptions

- Installing on [Framework Laptop 13](https://frame.work/products/laptop13-diy-intel-ultra-1)
- Using ethernet during installation
- Hardware SSD encryption doesn't work yet: [Issues enabling BitLocker hardware encryption](https://community.frame.work/t/issues-enabling-bitlocker-hardware-encryption-windows-encrypted-hard-drive-on-amd-7840/39415/66)

## Prerequisites

## 0. Prepare installer

_NOTE: If using PXE boot, prepare network bootloaders (e.g. netboot.xyz, iventoy, etc).__

1. Download Arch Linux image: https://archlinux.org/download/

2. Write downloaded image to USB drive (e.g. [balenaEtcher](https://etcher.balena.io/), [Ventoy](https://www.dwarmstrong.org/ventoy))

3. Boot target (Framework) laptop, and disable `Enforce secure boot` in BIOS.

4. Boot from installer (either USB drive, or PXE Boot - Arch Linux Installer).

NOTE: If no ethernet available, `iwctl` [can be used](https://wiki.archlinux.org/title/Network_configuration/Wireless) to establish wifi connection.

5. (Optional) Enable SSH to login to the target system from another laptop and be able to copy paste commands easily.
```
# systemctl start sshd.service
```

6. Set root password
```
# passwd
```

7. Find out assigned ip address
```
# ip a
```

8. Login to the target laptop from other device, as root
```
# ssh root@ip.address.obtained.above
```

## 1. Prepare install environment

1. Update system clock, and sync it to the hardware clock
```
# timedatectl list-timezones | grep London
# ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
# timedatectl set-ntp true
# timedatectl status
# date
# hwclock --systohc
```

2. Check mirror list, and select ones that make sense for you
```
# vim /etc/pacman.d/mirrorlist
```

3. Sync package database
```
# pacman -Sy
```

4. (Optional) Setup keyboard layout
```
# localectl list-keymaps | grep us
# loadkeys us
```

5. (Optional) Increase font size for high resolution monitors
```
# setfont ter-v24n
```
NOTE: Alternative fonts are available in `/usr/share/kbd/consolefonts`.

6. Check that we are in UEFI mode
```
# cat /sys/firmware/efi/fw_platform_size
```
or
```
# ls /sys/firmware/efi/efivars
```

## 2. Partition disk

Check disks:

```
# lsblk
```

1. Clean previous installation
```
# wipefs -af /dev/nvme0n1
# sgdisk --zap-all --clear /dev/nvme0n1
# partprobe /dev/nvme0n1
```

2. Fill disk with random data
- Create a temporary crypt device (e.g. "target")
```
# cryptsetup open --type plain -d /dev/urandom /dev/nvme0n1 target
```
- Fill the container with a stream of zeros using "dd":
```
# dd if=/dev/zero of=/dev/mapper/target bs=1M status=progress oflag=direct
```
- Remove the mapping
```
# cryptsetup close target
```

3. Partition disk

Use `sgdisk` to create partitions

- List partition type codes
```
# sgdisk --list-types
```

- Partition 1 - EFI partition (ESP) - size `512MiB`, code `ef00`
- Partition 2 - encrypted partition (LUKS) - remaining storage, code `8309`
- `-50G` # leaving 50GB at the end for the SSD to work with
```
# sgdisk -n 0:0:+512MiB -t 0:ef00 -c 0:esp /dev/nvme0n1
# sgdisk -n 0:0:-50GiB -t 0:8309 -c 0:luks /dev/nvme0n1
# partprobe /dev/nvme0n1
```

Check the new partition table
```
# sgdisk -p /dev/nvme0n1
```



## References
- Arch Linux Installation guide https://wiki.archlinux.org/title/Installation_guide
- A(rch) to Z(ram): Install Arch Linux with (almost) full disk encryption and BTRFS https://www.dwarmstrong.org/archlinux-install/
- BTRFS snapshots and system rollbacks on Arch Linux https://www.dwarmstrong.org/btrfs-snapshots-rollbacks/
- Create a USB stick with multiple Linux install images using Ventoy https://www.dwarmstrong.org/ventoy/
- Modern Arch linux installation guide https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae
- Install Arch on Framework laptop with BTRFS, encrypted swap and hibernate https://gist.github.com/RobFisher/abd9b2b9fca4194ac8df112715045b61
- Arch Linux with Xfce4 and i3 Window Manager Installation Guide https://github.com/silentz/arch-linux-install-guide/blob/main/README.md
- Arch Linux install with full disk encryption using LUKS2 - Logical Volumes with LVM2 - Secure Boot - TPM2 Setup https://github.com/joelmathewthomas/archinstall-luks2-lvm2-secureboot-tpm2
- Framework Laptop 13 https://wiki.archlinux.org/title/Framework_Laptop_13
- Arch Linux on the Framework Laptop 13 https://community.frame.work/t/arch-linux-on-the-framework-laptop-13/3843
- Arch Linux | GRUB https://wiki.archlinux.org/title/GRUB
- Comparing LUKS and LUKS2: A Comprehensive Analysis https://www.e2encrypted.com/posts/luks-vs-luks2/
- Should I use LUKS1 or LUKS2 for partition encryption? https://askubuntu.com/questions/1032546/should-i-use-luks1-or-luks2-for-partition-encryption
- dm-crypt/Encrypting an entire system https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system
- dm-crypt/Device encryption https://wiki.archlinux.org/title/Dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode
- luksDump shows lots of key slots enabled https://bbs.archlinux.org/viewtopic.php?id=175379
- Changing a LUKS Passphrase https://www.baeldung.com/linux/luks-change-passphrase
- Self-encrypting drives https://wiki.archlinux.org/title/Self-encrypting_drives
- Cryptsetup 2.7.0 Release Notes https://mirrors.edge.kernel.org/pub/linux/utils/cryptsetup/v2.7/v2.7.0-ReleaseNotes
- dm-crypt/Drive preparation https://wiki.archlinux.org/title/Dm-crypt/Drive_preparation
- Cryptsetup https://gitlab.com/cryptsetup/cryptsetup/-/wikis/FrequentlyAskedQuestions#2-setup
- Managing Partitions with sgdisk https://fedoramagazine.org/managing-partitions-with-sgdisk/
- 

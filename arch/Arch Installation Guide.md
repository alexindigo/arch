# Arch Installation Guide

- [Key Points](#key-points)
- [Assumptions](#assumptions)
- [Prerequisites](#prerequisites)
- [0. Prepare installer](#0-prepare-installer)
- [1. Prepare install environment](#1-prepare-install-environment)
- [2. Partition disk](#2-partition-disk)
- [3. Disk Encryption](#3-disk-encryption)
- [4. Format partitions](#4-format-partitions)
- [5. Mount volumes](#5-mount-volumes)
- [6. Install Arch](#6-install-arch)
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

...

## 0. Prepare installer

_NOTE: If using PXE boot, prepare network bootloaders (e.g. netboot.xyz, iventoy, etc)._

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

- Partition 1 - EFI partition - size `1GiB`, code `ef00`
- Partition 2 - encrypted partition (LUKS) - remaining storage, code `8309`
- `-50G` # leaving 50GB at the end for the SSD to work with
```
# sgdisk -n 0:0:+1GiB -t 0:ef00 -c 0:esp /dev/nvme0n1
# sgdisk -n 0:0:-50GiB -t 0:8309 -c 0:luks /dev/nvme0n1
# partprobe /dev/nvme0n1
```

Check the new partition table
```
# sgdisk -p /dev/nvme0n1
```

## 3. Disk Encryption

1. Create LUKS encrypted container. **You will need to create and enter (twice) encryption passphrase**
```
# cryptsetup -v -y luksFormat /dev/nvme0n1p2
```

2. Open created container
```
# cryptsetup luksOpen /dev/nvme0n1p2 crypt
```
** – `crypt` is the name of the encrypted container after opening it. The decrypted container is now available at `/dev/mapper/crypt`.

## 4. Format partitions

1. Format EFI (boot) partition
```
# mkfs.vfat -F32 -n EFI /dev/nvme0n1p1
```
** – `EFI` is the partition name

2. Format encrypted partition
```
# mkfs.btrfs -L CRYPT /dev/mapper/crypt
```
** – `CRYPT` is the filesystem label

3. Mount root filesystem to create subvolumes
```
# mount /dev/mapper/crypt /mnt
```

4. Create BTRFS subvolumes

Separate data (for backup/restore purposes) into individual subvolumes.

 - Create "top level" subvolume (make it easier to change subvolumes in the future by using separate `@` subvolume as root `/`)

```
# btrfs subvolume create /mnt/@
```

- Create swap subvolume
```
# btrfs subvolume create /mnt/@swap
```

- Create rest of subvolumes (pick ones matching your needs) for the following mount points:
- `@cache`: `/var/cache`
- `@home`: `/home`
- `@log`: `/var/log`
- `@snapshots`: `/.snapshots`
- `@tmp`: `/var/tmp`
- `@vms`: `/var/lib/libvirt`

```
# btrfs subvolume create /mnt/@cache
# btrfs subvolume create /mnt/@home
# btrfs subvolume create /mnt/@log
# btrfs subvolume create /mnt/@snapshots
# btrfs subvolume create /mnt/@tmp
# btrfs subvolume create /mnt/@vms
```

5. Unmount root filesystem, to re-mount all subvolumes to the right places
```
# umount /mnt
```

## 5. Mount volumes

1. Prepare shared mount options:

Recommended options for SSD/NVME:
> `noatime` – don't write access time for files;
> `compress-force=zstd:1` – less compression for higher speed;
> `space_cache=v2` – in memory cache;

```
# export sv_opts="rw,noatime,compress-force=zstd:1,space_cache=v2"
```

2. Mount the root subvolume `@`, so we can mount all other subvolumes under this root volume
```
# mount -o ${sv_opts},subvol=@ /dev/mapper/crypt /mnt
```

3. Mount boot partition
```
# mkdir -p /mnt/boot
# mount /dev/nvme0n1p1 /mnt/boot
```

4. Create mount points for the the rest of subvolumes, including `swap` 
```
# mkdir -p /mnt/{.snapshots,home,swap,var/cache,var/lib/libvirt,var/log,var/tmp}
```

5. Mount subvolumes
```
# mount -o ${sv_opts},subvol=@snapshots /dev/mapper/crypt /mnt/.snapshots
# mount -o ${sv_opts},subvol=@home /dev/mapper/crypt /mnt/home
# mount -o ${sv_opts},subvol=@swap /dev/mapper/crypt /mnt/swap
# mount -o ${sv_opts},subvol=@cache /dev/mapper/crypt /mnt/var/cache
# mount -o ${sv_opts},subvol=@vms /dev/mapper/crypt /mnt/var/lib/libvirt
# mount -o ${sv_opts},subvol=@log /dev/mapper/crypt /mnt/var/log
# mount -o ${sv_opts},subvol=@tmp /dev/mapper/crypt /mnt/var/tmp
```

6. Check new structure
```
# lsblk
```

## 6. Install Arch

0. Set package mirrors

- Synchronize package databases
```
# pacman -Syy
```

- Generate mirror list (top 5 by speed in US/Canada)
```
# reflector --verbose --protocol https --latest 5 --sort rate --country US --country Canada --save /etc/pacman.d/mirrorlist
```

1. Install base system

** - _For AMD use `amd-ucode` microcode package, instead of `intel-ucode` used for intel._

> if you want to continue setting up laptop by sshing into it, add `openssh` to the list.

```
# pacstrap /mnt base intel-ucode btrfs-progs linux linux-firmware git vim sudo
```

2. Update file system table

```
# genfstab -U -p /mnt >> /mnt/etc/fstab
```

3. Chroot into the base system to configure

```
# arch-chroot /mnt /bin/bash
```

4. Timezone

Set desired timezone and update the system clock

```
# ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
# hwclock --systohc
```

5. Hostname

** – _`framework` is example laptop name_

```
# echo "framework" > /etc/hostname
```

- Add matching entries to `/etc/hosts`

```
# cat > /etc/hosts <<EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   framework.localdomain framework
EOF
```

** - Details on `.localdomain`: https://bbs.archlinux.org/viewtopic.php?id=156064

7. Locale

Set `en_US.UTF-8` locale

```
# export locale="en_US.UTF-8"
# sed -i "s/^#\(${locale}\)/\1/" /etc/locale.gen
# echo "LANG=${locale}" > /etc/locale.conf
# locale-gen
```

- (Optional) if needed set key mapping for non-US locales/keyboards
_Details: https://man.archlinux.org/man/vconsole.conf.5

```
# echo "KEYMAP=us" >> /etc/vconsole.conf
```

8. Root password

```
# passwd
```

9. Add first (admin) user

- Create a user account, `alex` is example user name

```
# useradd -m -G wheel -s /bin/bash alex
# passwd alex
```

- Enable `wheel` group access for `sudo`

```
# sed -i "s/# %wheel ALL=(ALL:ALL) ALL/%wheel ALL=(ALL:ALL) ALL/" /etc/sudoers
```

## Setup boot

```
# pacman -Syu grub efibootmgr mtools dosfstools base-devel linux-headers
```

- Get UUID of boot device and append it to end of file for cutting and pasting:

```
# blkid
# blkid | grep n1p2 | cut -d\" -f 2
# blkid | grep n1p2 | cut -d\" -f 2 >> /etc/default/grub
# vim /etc/default/grub
```

- Edit `GRUB_CMDLINE_LINUX_DEFAULT` to look like this; you can cut the UUID from the end of the file by pressing `v` to go into visual mode, moving cursor to the end, and then pressing `d` to cut. Later you can paste it with the `p` command.

```
"loglevel=3 quiet acpi_osi=\"Windows 2020\" mem_sleep_default=deep cryptdevice=UUID=PASTED-UUID:crypt:allow-discards root=/dev/mapper/crypt"
```



## Temporary

Install further down the line, maybe
```
# base-devel bash-completion cryptsetup htop man-db mlocate neovim networkmanager openssh pacman-contrib pkgfile reflector terminus-font tmux
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
- Snapper - backup (snapshots) tool https://wiki.archlinux.org/title/Snapper
- `man mount` https://www.man7.org/linux/man-pages/man8/mount.8.html
- What BTRFS subvolume mount options should I use ? https://www.reddit.com/r/btrfs/comments/shihry/what_btrfs_subvolume_mount_options_should_i_use/
- btrfs(5) man https://btrfs.readthedocs.io/en/latest/btrfs-man5.html
- `.localdomain` https://bbs.archlinux.org/viewtopic.php?id=156064
- `Re: localhost.localdomain` https://lists.debian.org/debian-devel/2005/10/msg00559.html
- 

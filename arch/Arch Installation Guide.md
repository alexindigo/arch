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
- [7. Setup boot image](#7-setup-boot-image)
- [8. Prepare for the first boot](#8-prepare-for-the-first-boot)
- [9. After first boot](#9-after-first-boot)
- [10. Encrypted Swap](#10-encrypted-swap)
- [11. Hibernation](#11-hibernation)
- [12. Linux LTS kernel](#12-linux-lts-kernel)
- [13. Tuning](#13-tuning)
- [14. Snapshots](#14-snapshots)
- [References](#references)

## Key Points

- BTRFS
- Encryption (LUKS) + encrypted swap
- Hibernation
- Backup kernel
- Snapshots, auto-update GRUB
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

5. (Optional) If no plugged in to power, check battery status
```
# cat /sys/class/power_supply/BAT1/capacity
```

6. (Optional) Enable SSH to login to the target system from another laptop and be able to copy paste commands easily.
```
# systemctl start sshd.service
```

7. Set root password
```
# passwd
```

8. Find out assigned ip address
```
# ip a
```

9. Login to the target laptop from other device, as root
```
$ ssh root@ip.address.obtained.above
```

## 1. Prepare install environment

1. Update system clock, and sync it to the hardware clock
```
# timedatectl list-timezones | grep Los_Angeles
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
# setfont ter-v28n
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
** - If the directory does not exist, the system is booted in BIOS mode.

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
# sgdisk -n 0:0:+1GiB -t 0:ef00 -c 0:boot /dev/nvme0n1
# sgdisk -n 0:0:-50GiB -t 0:8309 -c 0:vault /dev/nvme0n1
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

## 7. Setup boot image

0. Install dependencies
```
# pacman -Syu grub efibootmgr mtools dosfstools base-devel linux-headers
```

1.  Install GRUB
```
# grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

- Verify that a GRUB entry has been added to the UEFI bootloader:
```
# efibootmgr
```

- (Optional) You may want to adjust boot order, using `efibootmgr -o`, **make sure to use correct values** from previous output.
```
# efibootmgr -o 0001,2001,2002,2003
```

2. Prepare GRUB config template

- Get UUID of boot device and append it to end of file for cutting and pasting:

```
# blkid -s UUID -o value /dev/nvme0n1p2 >> /etc/default/grub

# vim /etc/default/grub
```

- Edit `GRUB_CMDLINE_LINUX_DEFAULT` to look like this; you can cut the UUID from the end of the file by pressing `v` to go into visual mode, moving cursor to the end, and then pressing `d` to cut. Later you can paste it with the `p` command.

```
"loglevel=3 quiet acpi_osi=\"Windows 2020\" mem_sleep_default=deep cryptdevice=UUID=CUT-N-PASTED-UUID-FROM-BOTTOM-OF-FILE:crypt:allow-discards root=/dev/mapper/crypt"
```

3. Set up GRUB config
```
# grub-mkconfig -o /boot/grub/grub.cfg
```

4. Set up boot image

- Set `MODULES` and `HOOKS` in `/etc/mkinitcpio.conf`:
```
# vim /etc/mkinitcpio.conf
```

- `MODULES`
```
MODULES=(btrfs)
```

- `HOOKS`
```
HOOKS=(base udev keyboard autodetect microcode keymap consolefont modconf block encrypt filesystems fsck)
```

Hook ordering:

| Hook             | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `base`           | Sets up all initial directories and installs base utilities and libraries. Always keep this hook as the first hook unless you know what you are doing, as it provides critical busybox init when not using **systemd** hook.                                                                                                                                                                                                                                                                                                     |
| `udev`           | Starts the udev daemon and processes uevents from the kernel; creating device nodes. As it simplifies the boot process by not requiring the user to explicitly specify necessary modules, using it is recommended.                                                                                                                                                                                                                                                                                                               |
| `keyboard`       | Adds the necessary modules for keyboard devices. Use this if you have a USB or serial keyboard and need it in early userspace (either for entering encryption passphrases or for use in an interactive shell). For systems that are booted with different hardware configurations (e.g. laptops with external keyboard vs. internal keyboard), **this hook needs to be placed before autodetect** in order to be able to use the keyboard at boot time, for example to unlock an encrypted device when using the `encrypt` hook. |
| **`autodetect`** | Shrinks your initramfs to a smaller size by creating a whitelist of modules from a scan of sysfs. Be sure to verify included modules are correct and none are missing. This hook must be run before other subsystem hooks in order to take advantage of auto-detection. Any hooks placed before 'autodetect' will be installed in full.                                                                                                                                                                                          |
| `microcode`      | Prepends an uncompressed initramfs image with early microcode update files for Intel and AMD processors. If the autodetect hook runs before this hook, it will only add early microcode update files for the processor of the system the image is built on. This also allows you to drop the microcode `initrd` lines from your boot configuration as they are now packed together with the main initramfs image. |
| `keymap`         | Adds the specified console keymap(s) from `/etc/vconsole.conf` to the initramfs. If you use system encryption, **especially full-disk encryption, make sure you add it before the `encrypt` hook**.                                                                                                                                                                                                                                                                                                                              |
| `consolefont`    | Adds the specified console font from `/etc/vconsole.conf` to the initramfs.                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `modconf`        | Includes modprobe configuration files from `/etc/modprobe.d/` and `/usr/lib/modprobe.d/`                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `block`          | Adds block device modules. If the `autodetect` hook runs before this hook, it will only add modules for block devices used on the system.                                                                                                                                                                                                                                                                                                                                                                                        |
| `encrypt`        | Adds the `dm_crypt` kernel module and the `cryptsetup` tool to the image. At runtime detects and unlocks an encrypted root partition. This must be placed **before** `filesystems`                                                                                                                                                                                                                                                                                                                                               |
| `filesystems`    | This includes necessary file system modules into your image. This hook is required unless you specify your file system modules in `MODULES`.                                                                                                                                                                                                                                                                                                                                                                                     |
| `fsck`           | Adds the fsck binary and file system-specific helpers to allow running fsck against your root device (and `/usr` if separate) prior to mounting. If added after the **autodetect** hook, only the helper specific to your root file system will be added. Usage of this hook is **strongly** recommended.                                                                                                                                                                                                                        |

- Recreate the initramfs image [for `linux` preset](https://wiki.archlinux.org/title/Mkinitcpio#Manual_generation)
```
# mkinitcpio -p linux
```

## 8. Prepare for the first boot

1. Enable auto-start FS trimmer

- Periodic TRIM optimizes performance on SSD storage, enable a weekly task that discards unused blocks
```
# systemctl enable fstrim.timer
```

2. (Optional) Install convenience tools, to use post reboot:
```
# pacman -Sy man-db man-pages openssh networkmanager
```

3. (Optional) Enable auto-start for network manager if chosen above
```
# systemctl enable NetworkManager
```

4. Exit chroot and reboot
```
# exit
# umount -R /mnt
# reboot
```

- Remove the USB stick. Type the LUKS password when prompted. Log in with your user created above.

## 9. After first boot

0. (Optional) you may need to connect to WIFI
```
$ sudo nmtui
```

2. (Optional) Enable SSH to continue setting up new laptop from another machine
```
$ sudo systemctl start sshd.service
$ ip a
```

- SSH from another machine as newly created sudo user (e.g. `alex`)

2. Check errors
```
$ systemctl --failed
$ journalctl -p 3 -xb
```

- investigate the errors and continue to setting up swap

## 10. Encrypted Swap

1. Create swap file (making couple gigs bigger than ram size)
```
$ sudo btrfs filesystem mkswapfile --size 98G /swap/swapfile
```

2. Make `swapfile` into swap
```
$ sudo swapon /swap/swapfile
```

5. Update `fstab`
```
# vim /etc/fstab
```
Add following line:
```
/swap/swapfile none swap defaults 0 0
```

6. Check results
```
# cat /proc/swaps
```

## 11. Hibernation

1. Add `resume` hook to the boot image
```
$ sudo vim /etc/mkinitcpio.conf
```

| Hook             | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ---------------- | -----------------------------------------------------------------------------------------------------------------------------------------------------------|
| `resume`          | Tries to resume from the "suspend to disk" state. See [Hibernation](https://wiki.archlinux.org/title/Hibernation "Hibernation") for further configuration. |

```
HOOKS=(base udev keyboard autodetect microcode keymap consolefont modconf block encrypt filesystems resume fsck)
```
** - `resume` needs to follow after `filesystems`

2. Rebuilt boot image
```
$ sudo mkinitcpio -p linux
```

3. Save resume offset
```
$ sudo btrfs inspect-internal map-swapfile -r /swap/swapfile
```
- returned value should be used as `resume_offset=XXXXXX` parameter to grub config

4. Get UUID for device with swapfile
```
$ findmnt -no UUID -T /swap/swapfile
```
- returned value should be used as `resume=UUID=SWAPFILE-UUID-DIFF-FROM-CRYPTUUID` parameter to grub config

5. Update bootloader
- Add retrived UUID to `GRUB_CMDLINE_LINUX_DEFAULT` line in `/etc/default/grub`
```
$ sudo vim /etc/default/grub
```
making it look like:
```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet acpi_osi=\"Windows 2020\" mem_sleep_default=deep cryptdevice=UUID=CUT-N-PASTED-UUID-FROM-BOTTOM-OF-FILE:crypt:allow-discards root=/dev/mapper/crypt resume=UUID=UUID=SWAPFILE-UUID-DIFF-FROM-CRYPTUUID resume_offset=XXXXXX"
```

6. Recreate GRUB config
```
$ sudo grub-mkconfig -o /boot/grub/grub.cfg
```

7. Reboot for GRUB changes to take effect
```
$ sudo reboot
```

8. Test hibernation
```
$ sudo systemctl hibernate
```

9. After Boot Check
```
$ uptime
```

## 12. Linux LTS kernel

1. Install the Long-Term Support (LTS) Linux kernel as a fallback option to Arch's default kernel
```
$ sudo pacman -S linux-lts
```

2. Recreate GRUB config
```
$ sudo grub-mkconfig -o /boot/grub/grub.cfg
```

3. Reboot and choose LTS kernel
```
$ sudo reboot
```

4. Check for LTS kernel 
```
$ uname -r
```

5. Reboot again and choose default (arch) kernel
```
$ sudo reboot
```

## 13. Tuning

1. Update packages
```
$ sudo pacman -Syu
```

2. Tune pacman
```
$ sudo vim /etc/pacman.conf
```
add `Misc Options`
```
Color
VerbosePkgLists
ParallelDownloads=5
```

3. Tune up GRUB config
```
$ sudo vim /etc/default/grub
```
- update/uncomment:
```
# remember the last entry you booted from
GRUB_DEFAULT=saved

# change timeout behavior, press ESC key to display menu
GRUB_TIMEOUT_STYLE=hidden

# make text of readable size
GRUB_GFXMODE=1024x768x32,800x600x32,auto

# enable saving the selected entry
GRUB_SAVEDEFAULT=true
```
-  Recreate GRUB config
```
$ sudo grub-mkconfig -o /boot/grub/grub.cfg
```
4. `/tmp` cleanup
```
$ sudo vim /etc/tmpfiles.d/tmp.conf
```
- add
```
# Cleaning up /tmp directory everytime system boots
D! /tmp 1777 root root 0
```
## 14. Snapshots

0. Prepare for manual installation, as `yabsnap`  isn't available via `pacman` yet.
```
$ sudo pacman -S base-devel git
$ mkdir Packages
$ cd Packages
```

1. Install `yabsnap` (Yet Another Btrfs Snapshotter) using git option

```
$ git clone https://github.com/hirak99/yabsnap
$ cd yabsnap
$ sudo scripts/install.sh
```

2. Configure `yabsnap`

- Create config files for each subvolume
```
$ sudo yabsnap create-config root
$ sudo yabsnap create-config home
$ sudo yabsnap create-config cache
$ sudo yabsnap create-config log
```

- Edit create config files
```
$ sudo vim /etc/yabsnap/configs/root.conf
$ sudo vim /etc/yabsnap/configs/home.conf
$ sudo vim /etc/yabsnap/configs/cache.conf
$ sudo vim /etc/yabsnap/configs/log.conf
```
Example:  https://github.com/hirak99/yabsnap?tab=readme-ov-file#config-file

- Add subvolumes mount points as `source`

`root.conf`:
```
source = /
keep_user = 5
```

`home.conf`:
```
source = /home
keep_user = 5
```

`cache.conf`:
```
source = /var/cache
keep_user = 5
```

`log.conf`:
```
source = /var/log
keep_user = 5
```

4. Check setup
```
$ yabsnap list
```

6. Enable service (auto snapshots)
```
$ sudo systemctl enable --now yabsnap.timer
```

7. Test snapshot
```
$ sudo yabsnap create --comment "Initial test snapshot, after configuring yabsnap"
$ yabsnap list
```

8. Integrate snapshots with GRUB via `grub-btrfs`

- install package
```
$ sudo pacman -S grub-btrfs
```

- (Optional) if `grub.cfg` location is different from default (`/boot/grub`), set the location of the directory containing the `grub.cfg` file in `/etc/default/grub-btrfs/config`
```
$ sudo vim /etc/default/grub-btrfs/config
```
Update:
```
GRUB_BTRFS_GRUB_DIRNAME="/boot/grub"
```

- Enable service (turn on auto update)
```
$ sudo systemctl start grub-btrfsd
$ sudo systemctl enable grub-btrfsd
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
- BTRFS docs: https://btrfs.readthedocs.io
- What BTRFS subvolume mount options should I use ? https://www.reddit.com/r/btrfs/comments/shihry/what_btrfs_subvolume_mount_options_should_i_use/
- btrfs(5) man https://btrfs.readthedocs.io/en/latest/btrfs-man5.html
- `.localdomain` https://bbs.archlinux.org/viewtopic.php?id=156064
- `Re: localhost.localdomain` https://lists.debian.org/debian-devel/2005/10/msg00559.html
- Mkinitcpio > Common hooks: https://wiki.archlinux.org/title/Mkinitcpio#Common_hooks
- Microcode initramfs packed together with the main initramfs in one file https://wiki.archlinux.org/title/Microcode#mkinitcpio
- Arch Wiki | Hibernation https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation
- Swap encryption with suspend-to-disk support https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption#With_suspend-to-disk_support
- How do you find the physical offset for a file in BTRFS? https://unix.stackexchange.com/questions/623859/how-do-you-find-the-physical-offset-for-a-file-in-btrfs
- BTFS | Swapfile https://btrfs.readthedocs.io/en/latest/Swapfile.html
- BTRFS | Swapfile for hibernation https://btrfs.readthedocs.io/en/latest/Swapfile.html#hibernation
- Hibernation on a LUKS Encrypted btrfs https://svw.au/guides/archbtw/hibernate-luks-btrfs-arch/
- Meaning of “watchdog did not stop!” Message at Shutdown in Linux https://www.baeldung.com/linux/watchdog-message-explained
- GRUB | Multiple entries https://wiki.archlinux.org/title/GRUB/Tips_and_tricks#Multiple_entries
- GRUB | Text size problem https://community.frame.work/t/solved-grub-text-size-problem/20036/9
- Arch | Silent boot https://wiki.archlinux.org/title/Silent_boot
- Arch | Plymouth https://wiki.archlinux.org/title/Plymouth
- Enabling hibernation on Arch Linux with LUKS LVM encryption using a swap partition https://gist.github.com/Iwwww/008ef082a52cc509d186889118412aa6
- Arch | BTRFS `/tmp` clean https://github.com/Zelrin/arch-btrfs-install-guide?tab=readme-ov-file#tmp-cleanup
- BTRFS snapshots and system rollbacks on Arch Linux https://www.dwarmstrong.org/btrfs-snapshots-rollbacks/
- arch-btrfs-install-guide https://github.com/Zelrin/arch-btrfs-install-guide
- An Arch Linux Installation on a Btrfs Filesystem with Snapper for System Snapshots and Rollbacks https://www.ordinatechnic.com/distribution-specific-guides/Arch/an-arch-linux-installation-on-a-btrfs-filesystem-with-snapper-for-system-snapshots-and-rollbacks
- Snapper/BTRFS layout for easily restoring files, or entire system https://bbs.archlinux.org/viewtopic.php?id=194491
- Arch | Snapper https://wiki.archlinux.org/title/Snapper
- BTRFS | Managing snapshots https://archive.kernel.org/oldwiki/btrfs.wiki.kernel.org/index.php/SysadminGuide.html#Managing_snapshots
- decrypt_keyctl - multi device decryption https://github.com/gebi/keyctl_keyscript
- Snapper and grub-btrfs in Arch Linux https://www.lorenzobettini.it/2023/03/snapper-and-grub-btrfs-in-arch-linux/
- `yabsnap` - Yet Another Btrfs Snapshotter https://github.com/hirak99/yabsnap
- yabsnap: btrfs snapshot manager for Arch https://www.reddit.com/r/archlinux/comments/y10kyx/yabsnap_btrfs_snapshot_manager_for_arch/
- grub-btrfs https://github.com/Antynea/grub-btrfs
- 

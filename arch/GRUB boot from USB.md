# GRUB boot from USB

```
Add this to the file /etc/grub.d/40_custom, replacing UUID=XXXX-YYYY with the partition UUID (get UUID with command blkid)

menuentry "Boot from USB Drive" {
    set root=UUID=XXXX-YYYY
    linux /vmlinuz root=UUID=XXXX-YYYY ro quiet splash
    initrd /initrd.img
}
```

OR 

```
menuentry "Boot from LIVE USB Drive" {
   search --set=root --fs-uuid DRIVE_UUID
   linux ($root)/casper/vmlinuz boot=casper quiet splash --
   initrd ($root)/casper/initrd.lz
}
```

## References
- Customizing GRUB for Repeat USB Booting https://thelinuxcode.com/boot-usb-using-grub/
- Customize Grub to Get a Better Experience With Linux https://itsfoss.com/customize-grub-linux/
- Framework | Custom bios boot logo? https://community.frame.work/t/custom-bios-boot-logo/12197/4
- Multiboot USB Drive With GRUB2 Bootloader https://github.com/adi1090x/uGRUB
- Boot Multiple ISO from USB via GRUB2 https://pendrivelinux.com/boot-multiple-iso-from-usb-via-grub2-using-linux/
- Arch | GRUB/Tips and tricks https://wiki.archlinux.org/title/GRUB/Tips_and_tricks#GUI_configuration_tools
- GRUB | Boot menu entry examples https://wiki.archlinux.org/title/GRUB#Boot_menu_entry_examples
- Use Linux efibootmgr Command to Manage UEFI Boot Menu https://www.linuxbabe.com/command-line/how-to-use-linux-efibootmgr-examples
- How to add a GRUB2 menu entry for booting installed Ubuntu on a USB drive? https://askubuntu.com/questions/344125/how-to-add-a-grub2-menu-entry-for-booting-installed-ubuntu-on-a-usb-drive
- Arch | Silent boot https://wiki.archlinux.org/title/Silent_boot#Retaining_or_disabling_the_vendor_logo_from_BIOS
- 

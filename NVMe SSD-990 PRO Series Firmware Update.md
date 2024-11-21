# NVMe SSD-990 PRO Series Firmware Update

1. Install nvme tool
```
# pacman -Sy nvme-cli curl
```

2. Check current firmware version
```
# nvme fw-log /dev/nvme0n1
Firmware Log for device:nvme0n1
afi  : 0x1
frs1 : 0x3744584a51324234 (4B2QJXD7)
```

3. Check whether Slot 1 is read-only:
```
# nvme id-ctrl /dev/nvme0n1 -H | grep Firmware
  [9:9] : 0x1	Firmware Activation Notices Supported
  [4:4] : 0x1	Firmware Activate Without Reset Supported
  [3:1] : 0x3	Number of Firmware Slots
  [0:0] : 0	Firmware Slot 1 Read/Write
```

4. Grab image from manufacturer site (https://semiconductor.samsung.com/consumer-storage/support/tools/)
```
# curl -OL "https://download.semiconductor.samsung.com/resources/software-resources/Samsung_SSD_990_PRO_4B2QJXD7.iso"
```

5. Unpack firmare files
```
# bsdtar -xf Samsung_SSD_990_PRO_4B2QJXD7.iso initrd
# bsdtar -xf initrd root
```

6. Find firmware file
```
# ls -1 root/fumagician/*.enc
root/fumagician/4B2QJXD7.enc
root/fumagician/DSRD.enc
```
The first file is the firmware.

TBW, follow https://wiki.archlinux.org/title/Solid_state_drive/NVMe#Generic

## References
- Samsung Storage Firmware https://semiconductor.samsung.com/us/consumer-storage/support/tools/
- How to update Samsung NVME firmware without Windows? https://bbs.archlinux.org/viewtopic.php?id=297806
- Arch Linux Wiki on Samsung SSD https://wiki.archlinux.org/title/Solid_state_drive/NVMe#Samsung
- Archi Linux Wiki on Generic Firmware Update https://wiki.archlinux.org/title/Solid_state_drive/NVMe#Generic
- 
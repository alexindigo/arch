# Post-install Tuning

> This guide assumes you followed [Arch Installation Guide](Arch%20Installation%20Guide.md) steps.

## AUR (Arch User Repository)

> Assuming you have registered account (https://aur.archlinux.org/register).

1. Setup SSH Auth

- Create SSH AUR specific key
```
$ ssh-keygen -f ~/.ssh/aur_id_ed25519 -P ""
```

- Update SSH config to use new key for AUR
```
$ vim ~/.ssh/config
```
Add:
```
Host aur.archlinux.org
  IdentityFile ~/.ssh/aur_id_ed25519
  User aur
```

- Add **public** key to AUR profile in _My Account_.

- Test SSH AUR connection
```
$ ssh aur@aur.archlinux.org help
```

3. Adjust 
```
$ sudo vim /etc/makepkg.conf
```

Add:
```
MAKEFLAGS="--jobs=$(nproc)"
```

## Power management

1. Install `acpid`
```
$ sudo pacman -S acpid
$ sudo systemctl enable acpid
$ sudo systemctl start acpid
```

2. Install Laptop Mode Tools
```
$ cd ~/Packages
$ git clone ssh://aur@aur.archlinux.org/laptop-mode-tools.git
$ cd laptop-mode-tools
$ less PKGBUILD
$ makepkg --clean
$ makepkg --install
```

3. (Optional) vote for the package if you find it useful
```
$ ssh aur@aur.archlinux.org vote laptop-mode-tools
```

4. Update `laptop-mode-tools` configs
```
$ sudo vim /etc/laptop-mode/laptop-mode.conf
$ ls -la /etc/laptop-mode/conf.d/
```

- Check enabled modules
```
$ grep -r '^\(CONTROL\|ENABLE\)_' /etc/laptop-mode/laptop-mode.conf
$ grep -r '^\(CONTROL\|ENABLE\)_' /etc/laptop-mode/conf.d
```

5. Adjust ACPI events
```
$ sudo vim /etc/systemd/logind.conf
```

> `systemctl hybrid-sleep` suspends the system both to RAM and disk, so a complete power loss does not result in lost data. This mode is also called _suspend to both_.

- Update config:
```
[Login]
SleepOperation=suspend-then-hibernate hybrid-sleep suspend hibernate
HandlePowerKey=sleep
HandlePowerKeyLongPress=poweroff
HandleLidSwitch=sleep
HandleLidSwitchExternalPower=sleep
```

> Can be one of `ignore`, `poweroff`, `reboot`, `halt`, `kexec`, `suspend`, `hibernate`, `hybrid-sleep`, `suspend-then-hibernate`, `sleep`, `lock`, and `factory-reset`, `secure-attention-key`.

- Reload logind service
```
$ sudo systemctl reload systemd-login
```

## TODO:
- lock on hibernate and suspend
- lock on "display off" (laptop tools)

## References

- Arch User Repository https://wiki.archlinux.org/title/Arch_User_Repository
- ACPI events https://wiki.archlinux.org/title/Power_management#ACPI_events
- Arch | Laptop https://wiki.archlinux.org/title/Laptop
- Power management/Suspend and hibernate https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate
- Laptop Mode Tools https://wiki.archlinux.org/title/Laptop_Mode_Tools
- Arch | `acpid`  https://wiki.archlinux.org/title/Acpid
- Arch | AUR | Authentication https://wiki.archlinux.org/title/AUR_submission_guidelines#Authentication
- Laptop Mode Tools wiki https://github.com/rickysarraf/laptop-mode-tools/wiki
- `logind.conf.5` https://man.archlinux.org/man/logind.conf.5
- 
- 

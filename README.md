# archinstall
Installation of a Arch Linux desktop/laptop with encrypted disks

## Download ISO file

Grab the latest ISO file from https://archlinux.org/download/ and get it a USB drive. Use whatever method you like, personally I prefer dd.

## Boot the USB

To boot the USB use the method of how your system does it, usually with DEL, ESC, F2 or schmayby F12.

## Prepping the install

Setup pacman to use faster mirrors

```
reflector --latest 20 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```


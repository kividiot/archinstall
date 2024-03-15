# archinstall
Installation of a Arch Linux desktop/laptop with encrypted disks

## Download ISO file

Grab the latest ISO file from https://archlinux.org/download/ and get it a USB drive. Use whatever method you like, personally I prefer dd.

## Boot the USB

To boot the USB use the method of how your system does it, usually with DEL, ESC, F2 or schmayby F12.

## Prepping the install

Load keymap of your choosing, US peeps only eed use the freedom keymap already provided. For my part I use a Swedish keyboard.
```
loadkeys sv-latin1
```
Setup pacman to use faster mirrors
```
reflector --latest 20 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```
Install btrfs-progs
```
pacman -Sy
pacman -S btrfs-progs
```
Find the device name of your install target, usually an nvme drive these days, so assuming that. Will continue the guide with /dev/nvme0n1, adjust accordingly..

## Setting up the harddisk





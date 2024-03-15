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
Run gdisk to setup your drive. First partition will contain the boot partition (EF00 for efi), second will contain your Arch Linux install (8309 for Luks).
```
gdisk /dev/nvme0n1
> o
> n
> ENTER
> ENTER
> +2048M
> EF00
> n
> ENTER
> ENTER
> ENTER
> 8309
> w
```
Create the encrypted partition and setup LVM, using LVM for a separate swap partition to enable hybernation. You can change the "system" to something else, but remember to change it in furthers step if so. Also do remember your password..
```
cryptsetup luksFormat /dev/nvme0n1p2
cryptsetup open /dev/sdX2 cryptlvm
pvcreate /dev/mapper/cryptlvm
vgcreate system /dev/mapper/cryptlvm
```
Create the LVMs for the system.
```
lvcreate -L 32G system -n swap
lvcreate -l +100%FREE system -n root
```
Format the partitions
```
mkfs.vat -F32 /dev/nvme0n1p1
mkswap /dev/mapper/system-swap
mkfs.btrfs /dev/mapper/system-root
```
Create the subvolumes on the btrfs partition, you can adjust the subvolumes to your liking, but rebember to mount them correctly later.
```
mount /dev/mapper/system-root /mnt
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@root
btrfs su cr /mnt/@log
btrfs su cr /mnt/@cache
btrfs su cr /mnt/@tmp
umount /mnt
```
Mount the filesystems
```
swapon /dev/mapper/system-swap
mount -o defaults,x-mount.mkdir,noatime,compress=zstd,commit=120,subvol=@ /dev/nvme0n1p2 /mnt
mount -o defaults,x-mount.mkdir,noatime,compress=zstd,commit=120,subvol=@home /dev/nvme0n1p2 /mnt/home
mount -o defaults,x-mount.mkdir,noatime,compress=zstd,commit=120,subvol=@root /dev/nvme0n1p2 /mnt/root
mount -o defaults,x-mount.mkdir,noatime,compress=zstd,commit=120,subvol=@log /dev/nvme0n1p2 /mnt/var/log
mount -o defaults,x-mount.mkdir,noatime,compress=zstd,commit=120,subvol=@cache /dev/nvme0n1p2 /mnt/var/cache
mount -o defaults,x-mount.mkdir,noatime,compress=zstd,commit=120,subvol=@tmp /dev/nvme0n1p2 /mnt/tmp
mkdir /mnt/boot
mount /dev/nvme0np1 /mnt/boot
```

## Install the system
Bootstrap the system, you can replace vim with nano if you like or install both maybe..
```
pacstrap /mnt base base-devel linux linux-firmware linux-headers vim btrfs-progs cryptsetup
```


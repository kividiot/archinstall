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
Generate the fstab
```
genfstab -U /mnt >> /mnt/etc/fstab
```
Make some niceties before rebooting. 
```
arch-chroot /mnt
```
Replace Europe/Stockholm with your timezone. 
```
ln -s /usr/share/zoneinfo/Europe/Stockholm /etc/localtime
```
Save current clock to th hardware clock.
```
hwclock –systohc
```
Replace HOSTNAME with your desired hostname. 
```
echo HOSTNAME > /etc/hostname
```
Enable locales, in my case "en_US.UTF-8 UTF-8" and "sv_SE.UTF-8 UTF-8", you can see later how to set different LC_ values for localization.
```
vim /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```
Set your keymap to persist. In my case sv-latin1
```
echo "KEYMAP=sv-latin1" > /etc/vconsole.conf
```
Update your packages, and install some extras. Add wpa_supplicant for wifi.
```
pacman -Syu
pacman -S networkmanager lvm2 dialog
systemctl enable NetworkManager
```
Add a user
```
useradd -m -G wheel USERNAME
passwd USERNAME
```
Set root password
```
passwd
```
Edit /etc/sudoers file and remove the "# " infront of %wheel
```
vim /etc/sudoers
```
## Setting up the boot
Install microcode updates
```
For AMD: pacman -S amd-ucode
For intel: pacman -S amd-ucode
```
We will need to setup the HOOKS for the boot
```
vim /etc/mkinitcpio.conf

HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 btrfs filesystems resume fsck)
```
Apply the settings, there will be some "WARNING", we will sort them later in the guide.
```
mkinitcpio -p linux
```
Install the bootloader
```
bootctl install
```
To setup the entry for the loader we will need the UUID of the partition, well write the value to the file and copy it to the correct place
```
blkid /dev/nvme0n1p2 -o value -s UUID >  /boot/loader/entries/arch.conf 
```
Edit the file
```
vim /boot/loader/entries/arch.conf
```
It will just contain the UUID for now, edit it to look like and replacing THEUUIDHERE with the actual UUID, if you don't want Zswap enabled, remove everything after "quiet", if you want a verbose boot, remove "quiet". If you are running on an Intel CPU, replace the "initrd /amd-ucode.img" with "initrd /intel-ucode.img".
```
title Arch Linux
linux /vmlinuz-linux
initrd /amd-ucode.img
initrd /initramfs-linux.img
options cryptdevice=UUID=THEUUIDHERE:lvm:allow-discards rw resume=/dev/mapper/system-swap root=/dev/mapper/system-root rootflags=subvol=@ rootfstype=btrfs quiet zswap.enabled=1 zswap.compressor=zstd zswap.max_pool_percent=20 zswap.zpool=z3fold
```
## Finishing up
Leave the chroot environment
```
exit
```
Umount the filesystems
```
umount -R /mnt
```
And reboot
```
reboot
```
## Before we go further, use your normal account from now on
Enable pacman Color, ParallelDownloads and multilib support
```
sudo vim /etc/pacman.conf

#Color
to
Color

#ParallelDownloads = 5
to
ParallelDownloads = 5

#[multilib]
#Include = /etc/pacman.d/mirrorlist
to
[multilib]
Include = /etc/pacman.d/mirrorlist
```
Update system
```
sudo pacman -Syu
```
## Installing yay
For faster install, lets bump up how many cores we can use during install
```
sudo vim /etc/makepkg.conf

#MAKEFLAGS="-j16"
to
MAKEFLAGS="-jNUMBEROFCORESTOUSE"
```
From now on we will use yay to install, update and remove packages. yay works for both normal packages and AUR packages.
```
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```
## Installing nvidia drivers

## Installing a GNOME desktop
```
yay -S --needed wayland
yay -S --needed gdm
yay -S --needed xorg-xwayland xorg-xlsclients glfw-wayland
yay -S --needed gnome gnome-tweaks nautilus-sendto gnome-nettool gnome-usage gnome-multi-writer adwaita-icon-theme xdg-user-dirs-gtk fwupd arc-gtk-theme
```
If using a Nvidia graphics card also install
```
yay -S --needed egl-wayland
```
Enable gdm and reboot
```
systemctl enable gdm
reboot
```

## Installing some packages I like

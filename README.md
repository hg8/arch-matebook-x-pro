# Arch Linux on Huawei Matebook X Pro 2019

Installation of Arch Linux on the Huawei MateBook Pro X 2019 with Full Disk Encryption.

This was done with a new Huawei MateBook Pro X 2019 Intel Core i7-8565U, 8GB RAM, 512GB SSD. Delivered on 2019/05/19.


## Installation
### Update BIOS if possible

Boot into Windows 10 once, and let it go through it's whole setup so you can update the BIOS.

### Create an Arch Boot USB Key

Download the [Arch ISO](https://www.archlinux.org/download/)

Burned it to a [UBS Key](https://wiki.archlinux.org/index.php/USB_flash_installation_media) like so:

```
$ sudo dd bs=4M if=path/to/archlinux.iso of=/dev/sdx status=progress oflag=sync
```

If on Windows you can use [Rufus](http://rufus.ie/) which worked great everytimes I tried.

### Boot from USB

To access the Boot menu, first disable Secure Boot in the BIOS (Enter with <kbd>F12</kbd>). 

`Security Setting > Secure Boot > Disable`

Then hold down <kbd>F2</kbd> while booting to enter the BIOS device selection and choose your Arch USB.


### Make Fonts Readable

Make the console font larger so it's readable, we'll set a permanent font 
later:

`# setfont latarcyrheb-sun32`


### Connect to internet

Most of this next part is from the [Arch Install Guide](https://wiki.archlinux.org/index.php/Installation_guide)

`# wifi-menu` 

Check that it works, it may take a few seconds for networking to come up.

`# ping archlinux.org`

Update the system clock

`# timedatectl set-ntp true`

### Partition and Format

This install will use full disk encryption, with the exception of the EFI
boot partition. 

```
gdisk /dev/nvme0n1
```

```
GPT fdisk (gdisk) version 1.0.1

Command (? for help): o
This option deletes all partitions and creates a new protective MBR.
Proceed? (Y/N): Y

Command (? for help): n
Partition number (1-128, default 1): 
First sector (34-242187466, default = 2048) or {+-}size{KMGTP}: 
Last sector (2048-242187466, default = 242187466) or {+-}size{KMGTP}: +512M
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): EF00
Changed type of partition to 'EFI System'

Command (? for help): n
Partition number (2-128, default 2): 
First sector (34-242187466, default = 1050624) or {+-}size{KMGTP}: 
Last sector (1050624-242187466, default = 242187466) or {+-}size{KMGTP}: 
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/sda: 242187500 sectors, 115.5 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 9FB9AC2C-8F29-41AE-8D61-21EA9E0B4C2A
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 242187466
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         1050623   512.0 MiB   EF00  EFI System
   2         1050624       242187466   115.0 GiB   8300  Linux filesystem

Command (? for help): w
```


### Setup Disk Encryption

I chose the realtively simple LVM on LUKS setup combining instructions from:
[Encrypting_an_entire_system](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS)

There will be a single LUKS2 volume with LVM on top. LVM will then divide that volume into `root`, `home` and `swap`.

Setup and open the LUKS2 volume

```
# cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
# cryptsetup open /dev/nvme0n1p2 S3V0
```

Setup LVM, with swap at least as large as RAM to support hibernate

```
# pvcreate /dev/mapper/S3V0
# vgcreate archvg /dev/mapper/S3V0
# lvcreate -L 16G S3V0 -n swap
# lvcreate -L 64G S3V0 -n root
# lvcreate -l 100%FREE S3V0 -n home
```

Format the filesystems

```
# mkfs.vfat -F32 /dev/nvme0n1p1
# mkfs.ext4 /dev/S3V0/root
# mkfs.ext4 /dev/S3V0/home
# mkswap /dev/S3V0/swap
```

Mount the partitions

```
# mount /dev/S3V0/root /mnt
# mkdir /mnt/home
# mount /dev/S3V0/home /mnt/home
# swapon /dev/S3V0/swap
```

Mount the boot/ESP volume

```
# mkdir /mnt/boot
# mount /dev/nvme0n1p1 /mnt/boot
```

Install the base system

`# pacstrap /mnt base base-devel`

Generate the fstab

`# genfstab -U /mnt >> /mnt/etc/fstab`

### Basic Configuration

This is mostly straight from the [Arch Wiki Installation Guide](https://wiki.archlinux.org/index.php/Installation_guide)

Chroot to the new arch install

`# arch-chroot /mnt`

Set the timezone and hwclock

```
# ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
# hwclock --systohc
```

Setup Locales, uncomment any needed in /etc/locale.gen and generate
them

```
# locale-gen
# echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```

Set a hostname

`# echo marchbook >> /etc/hostname`

### Make the system bootable

Install Intel Microcode Updates, this will install an initrd image that 
we add to our boot loader config.

`# pacman -S intel-ucode`

Next setup systemd-boot

`bootctl --path=/boot install`

Get the encrypted volume UUID for use in the systemd-boot config.

`# blkid /dev/nvme0n1p2`

Add a menu entry for Arch and configure the loader :

`# vim /boot/loader/loader.conf`

```
timeout 0
default arch
entries 0
```

`# vim /boot/loader/entries/arch.conf`

```
title	Arch Linux
linux /vmlinuz-linux
initrd	/initramfs-linux.img
options cryptdevice=UUID=<your partition UUID>:lvm:allow-discards resume=/dev/mapper/S3V0-swap root=/dev/mapper/S3V0-root initrd=/intel-ucode.img rw
```

Add the `keyboard`, `encrypt` and `lvm2` hooks to `/etc/mkinitcpio.conf`: 

```
HOOKS=(base udev autodetect keyboard keymap consolefont modconf block encrypt lvm2 filesystems fsck)
```

Regnerate initramfs

`# mkinitcpio -p linux `

Install some packages so `wifi-menu` works after reboot.

`# pacman -S dialog wpa_supplicant`

Set the root password

`# passwd`

Reboot

```
# exit
# reboot
```

## Post Install Configuration
### Connect to internet

Get wifi back online after first boot. 

`# wifi-menu`

Get networking to come up automatically with `netctl`

```
# systemctl enable netctl-auto@wlp0s20f3.service
```

> netctl profiles will be started/stopped automatically as you move from the range of one network into the range of another network (roaming). 


### Create a regular user
```
# useradd -G wheel -m hugo
# passwd hugo
```

Then log out and switch to that user.

## Setup power saving

Enable TLP for powersaving
```
# pacman -S tlp tlp-rdw
# systemctl enable tlp.service
# systemctl enable tlp-sleep.service
# systemctl mask systemd-rfkill.service
# systemctl mask systemd-rfkill.socket
```

Install ethtool, lsb-release and smartmontools at the suggestion of tlp-stat

`# pacman -S ethtool lsb-release smartmontools`


### Get X11 Working

_This configuration is fitted to me, update it according to your needs_

Install X11, bspwm, and a few other nice things

```
# pacman -S bspwm sxhkd nvidia xorg-server xorg-font-util xorg-fonts-75dpi xorg-fonts-100dpi xorg-mkfontdir xorg-mkfontscale xorg-xdpyinfo xorg-xrandr xorg-xset bumblebee bbswitch termite firefox mesa xf86-video-intel xbindkeys xorg-xmodmap xorg-xrdb
```

Enable Bumblebee with bbswitch for Nvidia / Intel switching
```
# systemctl enable bumblebeed.service
# gpasswd -a $USER bumblebee
```

### Other useful app

Few apps I need :

```
# pacman -S zsh vim git compton xorg-xinit python-pip python2 light openssh pass polybar rofi randr xorg-xsetroot feh noto-fonts-cjk arc-gtk-theme thunar lxappearance vlc gvfs thunar-archive-plugin thunar-volman tumbler raw-thumbnailer gvfs-mtp gpicview xorg-xkill exa bat xss-lock xautolock autocutsel dunst ncdu chromium unzip zip p7zip pacman-contrib tldr xdg-user-dirs scrot xclip blueman

```

### Setup my dot files

Install my homshick setup [github.com/hg8/dotfiles](https://github.com/hg8/dotfiles)

### Install a AUR Helper

Install Trizen to build packages from Arch AUR

```
$ git clone https://aur.archlinux.org/trizen.git
$ cd trizen
$ makepkg -si
```

### Automatically startx at user login

Edit your `~/.profile` (`~/.zprofile` for zsh users) with:

```
$ cat .zprofile                  
if [[ ! $DISPLAY && $XDG_VTNR -eq 1 ]]; then
  exec startx
fi
```

### Make Pacman faster

Sort the pacman mirrors by speed
[Arch Wiki Sorting Mirrors](https://wiki.archlinux.org/index.php/mirrors#Sorting_mirrors)


## Various configurations

### Setup the Linux Console font for HiDPi

Set a readable console font:

`# pacman -S terminus-font`

Create `/etc/vconsole.conf` with contents:

`FONT=ter-132n`

### Setup user dirs

Setup basic user directories (Documents, Pictures, etc...)

```
sudo pacman -S xdg-user-dirs
xdg-user-dirs-update
```


### Configure the Trackpad

Info from: [Linux with a Macbook Touchpad Feel, Pt 2](https://williambharding.com/blog/linux-to-macbook/linux-with-a-macbook-touchpad-feel-pt-2/)
I'm using Synamptics based on the recommendations from the above link.
I was finding the trackpad behavior pretty annoying, given I spent most of my 
days on a Mac and have been using Macs for over a decade. In spite of Synaptics 
limited support going forward, it seems to work better so far.

Here is how I configured it.

```
# pacman -S xf86-input-synaptics
```

Then I created `/etc/X11/xorg.conf.d/30-synaptics.conf` with these contents:
```
Section "InputClass"
        Identifier "touchpad catchall"
        Driver "synaptics"
        MatchIsTouchpad "on"
        # Enabling tap-to-click is a perilous choice that begets needing to set up palm detection/ignoring. Since I am fine clicking my touchpad, I sidestep the issue by disabling tapping. 
        Option "TapButton1" "0"
        Option "TapButton2" "0"
        Option "TapButton3" "0"
	# Using negative values for ScrollDelta implements natural scroll, a la Macbook default. 
        Option "VertScrollDelta" "-80"
	Option "HorizScrollDelta" "-80"
        # https://wiki.archlinux.org/index.php/Touchpad_Synaptics has a very buried note about this option
	# tl;dr this defines right button to be rightmost 7% and bottommost 5%
	Option "SoftButtonAreas" "93% 0 95% 0 0 0 0 0"  
        MatchDevicePath "/dev/input/event*"
EndSection
```

### Hibernate on Low Battery

Suspend on low battery to '/etc/udev/rules.d/99-lowbat.rules'
```
$ cat /etc/udev/rules.d/99-lowbat.rules         
# Suspend the system when battery level drops to 5% or lower
SUBSYSTEM=="power_supply", ATTR{status}=="Discharging", ATTR{capacity}=="[0-5]", RUN+="/usr/bin/systemctl suspend"

```

### Time Sync

Setup time sync
[Arch Wiki Systemd-timesyncd](https://wiki.archlinux.org/index.php/Systemd-timesyncd)
Edit `/etc/systemd/timesyncd.conf` uncomment these lines:
```
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org 0.us.pool.ntp.org
```
Then start it:

`# timedatectl set-ntp true `



## Fixes

### 4.0 Surround Speakers

To fix Huawei Matebook X Pro Speakers on Linux follow this [guide](https://github.com/hg8/arch-matebook-x-pro-2019/blob/master/guide-fix-matebook-x-pro-speakers-linux.md)

### Volume and Screen Brighness Buttons Work
Instructions generally came from 
[Arch Wiki Xbindkeys](https://wiki.archlinux.org/index.php/Xbindkeys)

To set the backlight we need `light`

`# pacman -S light`

We'll need xbindkeys
```
# pacman -S xbindkeys
$ xbindkeys -d > ~/.xbindkeysrc
```

Here is an example on how to use it :

```
XF86AudioRaiseVolume
    pactl set-sink-volume @DEFAULT_SINK@ -1000


XF86AudioLowerVolume
    pactl set-sink-volume @DEFAULT_SINK@ -1000


XF86AudioMute
    pactl set-sink-mute @DEFAULT_SINK@ toggle
   
XF86MonBrightnessUp
    light -A 10
  

XF86MonBrightnessDown
    light -U 10
```

You can also use my script [brightness](https://github.com/hg8/dotfiles/blob/master/bin/brightness) and [volume](https://github.com/hg8/dotfiles/blob/master/bin/volume) to show notifcations on volume and brightness change unsing dunst

![2019-05-22-082913_373x102_scrot](https://user-images.githubusercontent.com/9076747/58159341-cf227e80-7c6b-11e9-90b0-59f394ac3e93.png)
![2019-05-22-082926_356x102_scrot](https://user-images.githubusercontent.com/9076747/58159343-cfbb1500-7c6b-11e9-9aad-7d8ba1d858b9.png)

## Maintenance 
### BIOS Updates

Huawei provides downloadable BIOS updates packaged for Windows. With some effort, these can be installed from Linux.

[The following method is available](https://github.com/nekr0z/linux-on-huawei-matebook-13-2019#bios-updates).

Note that it would maybe be safer (no messing up with UEFI boot record) to use a Win2Go USB key to update BIOS.

Will report after my personal tries.



(_based on [@kelp](kelp/arch-matebook-x-pro) work for the 2018 version_)

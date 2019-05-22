Table of Contents
=================

   * [Arch Install Notes](#arch-install-notes)
      * [Getting Started](#getting-started)
   * [Booting into the Base Install](#booting-into-the-base-install)
      * [Make an Arch Boot USB Key](#make-an-arch-boot-usb-key)
      * [Boot from USB](#boot-from-usb)
      * [Make Fonts Readable and Get Online](#make-fonts-readable-and-get-online)
      * [Partition and Format](#partition-and-format)
      * [Setup Disk Encryption](#setup-disk-encryption)
      * [Basic Configs for the New Install](#basic-configs-for-the-new-install)
      * [Make the system bootable](#make-the-system-bootable)
   * [Post Install Config](#post-install-config)
      * [Create a regular user](#create-a-regular-user)
      * [Setup power saving](#setup-power-saving)
      * [Get X11 Working](#get-x11-working)
      * [Setup my dot files, includes x11, nvim and zsh configs](#setup-my-dot-files-includes-x11-nvim-and-zsh-configs)
      * [Make Pacman faster](#make-pacman-faster)
      * [Setup the Linux Console](#setup-the-linux-console)
      * [Configure the Trackpad](#configure-the-trackpad)
      * [Hibernate on Low Battery](#hibernate-on-low-battery)
      * [Time Sync](#time-sync)
      * [Make Sound Work](#make-sound-work)
      * [Make the Volume and Screen Brighness Buttons Work](#make-the-volume-and-screen-brighness-buttons-work)
      * [TODO](#todo)

# Arch Install Notes
Installing Arch Linux on the Huawei MateBook Pro X 2019 with Full Disk Encryption.

This was done with a new Huawei MateBook Pro X 2019 Intel Core i7-8565U, 8GB RAM, 512GB SSD. Delivered on 2019/05/19.

## Getting Started
I booted into Windows 10 once, and let it go through it's whole setup
only so I could update the BIOS.

In the future BIOS updates can be done using [the following method](https://github.com/nekr0z/linux-on-huawei-matebook-13-2019#bios-updates).

Note that it would maybe be safer (no messing up with UEFI boot record) to use a Win2Go USB key to update BIOS.

Will report after my personal tries.

# Booting into the Base Install

## Make an Arch Boot USB Key
Download the [Arch ISO](https://www.archlinux.org/download/)

I burned it to a [UBS Key](https://wiki.archlinux.org/index.php/USB_flash_installation_media) like this:

```
$ sudo dd bs=4M if=path/to/archlinux.iso of=/dev/sdx status=progress oflag=sync

```

If on Windows you can use [Rufus](http://rufus.ie/) which worked great everytimes I tried.

## Boot from USB

To access the Boot menu, first disable Secure Boot in the BIOS (Enter with <kbd>F12</kbd>). 

`Security Setting > Secure Boot > Disable`

Then hold down <kbd>F2</kbd> while booting to enter the BIOS device selection and choose your Arch USB.


## Make Fonts Readable and Get Online

Make the console font larger so it's readable, we'll set a permanent font 
later:

`# setfont latarcyrheb-sun32`

Most of this next part is from the [Arch Install Guide](https://wiki.archlinux.org/index.php/Installation_guide)

`# wifi-menu` 

Check that it works, it may take a few seconds for networking to come up.

`# ping archlinux.org`

Update the system clock

`# timedatectl set-ntp true`

## Partition and Format

This install will use full disk encryption, with the exception of the EFI
boot partition. 

https://gist.github.com/heppu/6e58b7a174803bc4c43da99642b6094b

@todo


## Setup Disk Encryption
I chose the realtively simple LVM on LUKS setup combining instructions from:
[Encrypting_an_entire_system](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS)

There will be a single LUKS2 volume with LVM on top. LVM will then divide
that volume into root, home and swap.

Setup and open the LUKS2 volume

```
# cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
# cryptsetup open /dev/nvme0n1p2 cryptlvm
```

Setup LVM, with swap at least as large as RAM to support hibernate

```
# pvcreate /dev/mapper/cryptlvm
# vgcreate archvg /dev/mapper/cryptlvm
# lvcreate -L 16G archvg -n swap
# lvcreate -L 64G archvg -n root
# lvcreate -l 100%FREE archvg -n home
```

Format the filesystems

```
# mkfs.ext4 /dev/archvg/root
# mkfs.ext4 /dev/archvg/home
# mkswap /dev/archvg/swap
```

Mount the partitions

```
# mount /dev/archvg/root /mnt
# mkdir /mnt/home
# mount /dev/archvg/home /mnt/home
# swapon /dev/archvg/swap
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

## Basic Configs for the New Install
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

## Make the system bootable

Install Intel Microcode Updates, this will install an initrd image that 
we add to our boot loader config.

`# pacman -S intel-ucode`

Next setup systemd-boot

@todo

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

# Post Install Config

Get wifi back online after first boot. We'll change how this is configured
later.

`# wifi-menu`

## Create a regular user
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

Get networking to come up automatically with `netctl`
```
# systemctl enable netctl-auto@wlp0s20f3.service
```

> netctl profiles will be started/stopped automatically as you move from the range of one network into the range of another network (roaming). 

## Get X11 Working

Install X11, bspwm, and a few other nice things
```
# pacman -S bspwm sxhkd nvidia xorg-server xorg-font-util xorg-fonts-75dpi xorg-fonts-100dpi xorg-mkfontdir xorg-mkfontscale xorg-xdpyinfo xorg-xrandr xorg-xset bumblebee bbswitch termite firefox mesa xf86-video-intel xbindkeys xorg-xmodmap xorg-xrdb
```

Enable Bumblebee with bbswitch for Nvidia / Intel switching
```
# systemctl enable bumblebeed.service
# gpasswd -a $USER bumblebee
```

## Other useful app
Few apps I need :

```
# pacman -S zsh vim git compton xorg-xinit python-pip python2 light openssh pass polybar rofi randr xorg-xsetroot feh noto-fonts-cjk arc-gtk-theme thunar lxappearance vlc gvfs thunar-archive-plugin thunar-volman tumbler raw-thumbnailer gvfs-mtp gpicview xorg-xkill exa bat xss-lock xautolock autocutsel dunst ncdu chromium unzip zip p7zip pacman-contrib tldr xdg-user-dirs scrot xclip blueman

```
## Setup my dot files, includes x11, nvim and zsh configs
Install my homshick setup [github.com/hg8/dotfiles](https://github.com/hg8/dotfiles)

Install Trizen to build packages from Arch AUR

```
$ git clone https://aur.archlinux.org/trizen.git
$ cd trizen
$ makepkg -si
```

## Automatically startx at user login

Edit your `~/.profile` (`~/.zprofile` for zsh users) with:

```
$ cat .zprofile                  
if [[ ! $DISPLAY && $XDG_VTNR -eq 1 ]]; then
  exec startx
fi
```

## Make Pacman faster

Sort the pacman mirrors by speed
[Arch Wiki Sorting Mirrors](https://wiki.archlinux.org/index.php/mirrors#Sorting_mirrors)


## Setup the Linux Console

Set a readable console font:

`# pacman -S terminus-font`

Create `/etc/vconsole.conf` with contents:

`FONT=ter-132n`


## Configure the Trackpad

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

## Hibernate on Low Battery

Suspend on low battery to '/etc/udev/rules.d/99-lowbat.rules'
```
$ cat /etc/udev/rules.d/99-lowbat.rules         
# Suspend the system when battery level drops to 5% or lower
SUBSYSTEM=="power_supply", ATTR{status}=="Discharging", ATTR{capacity}=="[0-5]", RUN+="/usr/bin/systemctl suspend"

```

## Time Sync

Setup time sync
[Arch Wiki Systemd-timesyncd](https://wiki.archlinux.org/index.php/Systemd-timesyncd)
Edit `/etc/systemd/timesyncd.conf` uncomment these lines:
```
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org 0.us.pool.ntp.org
```
Then start it:

`# timedatectl set-ntp true `


## Make Sound Work

This took some experimenting. At first all the speakers didn't work
and it sounded horrible.

1. Install needed packages

   `$ sudo pacman -S alsa-utils pulseaudio pulseaudio-alsa alsa-tools pavucontrol pulsemixer`

2. Use `hdajackretask` as root to apply to correct config

   `$ sudo hdajackretask`

3. Select `Realtek ALC256` codec on the top.

4. Check the `Show unconnected pins` and `Advanced overrides` options on right.

5. Apply the following configuration for `PIN ID: 0x14`

   ![2019-05-22-081324_1261x347_scrot](https://user-images.githubusercontent.com/9076747/58158269-9e414a00-7c69-11e9-824b-fc08df81903e.png)
   
6. Apply the following configuration for `PIN ID: 0x1b`

   ![2019-05-22-081708_1284x360_scrot](https://user-images.githubusercontent.com/9076747/58158467-042dd180-7c6a-11e9-9dca-8780d98f6a6a.png)
   
7. Select <kbd>Install boot override</kbd>

8. Since we have full disk encryption we need to edit our `/etc/mkiniticpio.conf` with:

   `FILES=(/usr/lib/firmware/hda-jack-retask.fw)`
   
9. Recreate your initramfs:

   `sudo mkinitcpio -P`
   
10. Reboot

11. Open `pavucontrol`, navigate to `Configuration` and under `Profile` dropdown, select `Analog Surround 4.0 Output`.

([Source](https://www.reddit.com/r/MatebookXPro/comments/8z4pv7/fix_for_the_2_out_of_4_speakers_issue_on_linux/) & [Source](https://aymanbagabas.com/2018/07/23/archlinux-on-matebook-x-pro.html))



## Make the Volume and Screen Brighness Buttons Work
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



(_based on [@kelp](kelp/arch-matebook-x-pro) work for the 2018 version_)

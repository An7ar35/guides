# Arch Linux installation guide for MacBook Pro Retina (_mid-2015 model_)

# Table of Contents
1. [Forewords](#forewords)
2. [Machine Specs](#machine-specs)
3. [Results](#results)
4. [Pre-Installation](#pre-installation)
    1. [Preparations](#preparations)
    1. [Making a bootable USB](#making-a-bootable-usb)
5. [Installation](#installation)
    1. [Booting from the USB stick](#booting-from-the-usb-stick)
    2. [Preliminary setup](#preliminary-setup)
    3. [Partitioning and Crypto volume setup](#partitioning-and-crypto-volume-setup)
    4. [System Setup I: Base System Installation](#system-setup-i:-base-system-installation)
    5. [System Setup II: Hardware and Tools](#system-setup-ii:-hardware-and-tools)
    6. [System Setup III: Desktop](#system-setup-iii:-desktop)
    7. [System Setup IV: Applications](#system-setup-iv:-applications)
6. [Bugs](#bugs)
7. [References](#references)


# Forewords

After lots of reading, searching, experimenting, furious late-night shell command typing :neckbeard: and 
do-overs here are the results of replacing OSX with Arch Linux (KDE) on a mid-2015 MacBook Pro. 

This guide is a record of my experience and results. I'm leaving this out there for anyone that
might find this useful.

I would also advise backing up your drive using a complete bit-to-bit cloning process (unlike me...
:cold_sweat:) so that you retain a copy of everything including the recovery partition on the Mac. 
It's better to be safe...

# Machine Specs

|           | Description |
|:---------:|:------|
| Display   | 15.4" LED-backlit Retina display with IPS technology; 2880-by-1800 native resolution at 220 pixels per inch with support for millions of colors |
| Processor | 2.5GHz quad-core Intel Core i7 processor (Turbo Boost up to 3.7GHz) with 6MB shared L3 cache |
| RAM       | 16GB of 1600MHz DDR3L memory |
| GPU       | Intel Iris Pro Graphics, <br>AMD Radeon R9 M370X with 2GB of GDDR5 memory |
| Storage   | 512GB PCIe-based flash |
| Webcam    | 720p FaceTime HD camera |
| Network   | 802.11ac Wi‑Fi wireless networking; IEEE 802.11a/b/g/n compatible, <br>Bluetooth 4.0 wireless technology |

# Results

//TODO

//Heating issues with mbp, fan control and turning off boost on the processor to keep things
reasonable temperature-wise.

# Pre-Installation

## Preparations

1. __Backup drive__ (complete drive clone preferable).
2. Make a copy of the colour profile file(s) on the mac located in 
   `/Library/ColorSync/Profiles/Displays/*` to a USB stick. It will be useful later on Linux.

## Making a bootable USB

#### From Linux

1. $ `dd if=archlinux.iso of=/dev/sdX bs=16M && sync`
	> Where `X` is your target USB drive letter (use `lsblk` for an overview of all connected 
	  drives to find out)
  	  
#### From a Mac

1. $ `diskutil unmountDisk diskX`
    > Where `X` is your target USB drive number (use `diskutil list` for an overview of all 
      connected drives to find out).
2. $ `sudo dd if=/Users/$USERNAME/Downloads/archlinux.iso of=/dev/diskX bs=16M`
    > Replace $USERNAME with your username on the mac and replace `X` with the USB drive's number.

# Installation

## Booting from the USB stick

Simply plug in the USB in the MBP and press the`Alt` (a.k.a. 'options') key during start up
to reach the boot menu.

## Preliminary setup

1. $ `loadkeys uk`
  	> Load your keyboard layout. Replace `uk` with whichever you have on your machine.
2. $ `wifi-menu`
  	> Connect up to the wifi network. On the MPB Pro 2015 the wifi gets detected and works 
      out of the box at this stage. :thumbsup:

## Partitioning and Crypto volume setup

The assumption here is that the hard drive where everything will be installed to is `/dev/sda`. The 
partition structure will thus be:  
`/dev/sda1` for the EFI partition,  
`/dev/sda2` for the Boot partition and  
`/dev/sda3` for the encrypted volume where the root, home and swap will be located.  

1. $ `cgdisk /dev/sda`
    > __Partition layout with just Linux__
    >
    > *Warning: this will erase every thing including the recovery partition for OSX. 
       No easy way to go back after this without a bit-to-bit HDD clone* 
    >
    > 1. 100M partition 'EFI' (Hex #`ef00`)
    > 2. 250M partition 'Boot' (Hex #`8300`)
    > 3. 100% remainder of the space for the crypto volume (Hex #`8300`)
    >
    > The root and home partition will be created later in the crypto volume 
2. $ `cryptsetup --verbose --verify-passphrase --cipher aes-xts-plain64 --key-size 512 --hash sha512 
--iter-time 5000 --use-random luksFormat /dev/sda3`
    > Sets up the crypto volume. 
3. $ `cryptsetup open --type luks /dev/sda2 mctoasty` 
    > `mctoasty` is the mounted name of the crypto volume. You can choose your own.

#### Create crypto volume partitions

1. $ `pvcreate /dev/mapper/mctoasty`
2. $ `vgcreate vg0 /dev/mapper/mctoasty`
3. $ `lvcreate --size 16G vg0 --name swap`
4. $ `lvcreate --size 50GB vg0 --name root` 
    > Adjust root's size as required.
5. $ `lvcreate -l +100%FREE vg0 --name home`

#### Formatting all the partitions

1. $ `mkfs.vfat -F32 /dev/sda1`
    > EFI partition
2. $ `mkfs.ext2 /dev/sda2`
    > Boot partition
3. $ `mkswap /dev/mapper/vg0-swap`
4. $ `mkfs.ext4 /dev/mapper/vg0-root`
5. $ `mkfs.ext4 /dev/mapper/vg0-home`

#### Mount all the partitions

1. $ `mount /dev/mapper/vg0-root /mnt`
2. $ `swapon /dev/mapper/vg0-swap`
3. $ `mkdir /mnt/boot`
4. $ `mount /dev/sda2 /mnt/boot`
5. $ `mkdir /mnt/boot/efi`
6. $ `mount /dev/sda1 /mnt/boot/efi`
7. $ `mkdir /mnt/home`
8. $ `mount /dev/mapper/vg0-home /mnt/home`

## System Setup I: Base System Installation

1. $ `pacstrap /mnt base base-devel grub-efi-x86_64 git efibootmgr bash-completion dialog wpa_supplicant`
    > Base packages along with GRUB and the EFI boot manager, git (will be useful later),
      bash completion and the stuff needed to keep the Wifi device in working order after reboot.  

#### Generate fstab

1. $ `genfstab -pU /mnt >> /mnt/etc/fstab`
    > __Important!__
    >
    > This generates the fstab. i.e.: It saves our mounted partitions and swap configuration 
      for persistence after a reboot. If you miss this step you'll have to reboot with the USB, 
      mount everything again and then generate the fstab file. :unamused:
2. $ `arch-chroot /mnt /bin/bash`

#### Setup system clock

1. $ `ln -s /usr/share/zoneinfo/$ZONE/$REGION /etc/localtime`
    > Replace $ZONE with yours from `/usr/share/zoneinfo/`  
      Replace $REGION with yours from `/usr/share/zoneinfo/$ZONE/`  
      e.g.: `/usr/share/zoneinfo/Europe/London`

2. $ `hwclock --systohc --utc`

#### Hostname

1. $ `echo $HOSTNAME > /etc/hostname`
2. $ `nano /etc/hosts`
    > Add the hostname at the end of each of the relevant lines in this file for completion's 
      sake. Replace $HOSTNAME by the hostname you want to computer to have.

#### Basic fonts

1. $ `pacman -S terminus-font ttf-dejavu ttf-liberation`
    > The terminus font will be used to make the console font more 
      [readable](https://wiki.archlinux.org/index.php/HiDPI#Linux_console) on the HiDPI display.

#### Locale

1. $ `locale-gen`
    > Generates the locale file
2. $ `nano /etc/local.conf`
    > Edit the locale configuration and delete the hash in front of the desired locale.
3. $ `local-gen`
    > Needs to run again to apply the changes made in (2).
4. $ `nano /etc/locale.conf`
    > To set permanent locale settings add the lines:   
    > ```bash
    > LANG=en_GB.UTF-8  
    > LANGUAGE=en_GB  
    > LC_ALL=C
    > ```

#### Console keymap and font

To see a list of all available keymaps: `find /usr/share/kbd/keymaps/ -type f | more`  
To see a list of all installed console fonts: `ls /usr/share/kbd/consolefonts/`  
For font maps check the [Arch Wiki](https://wiki.archlinux.org/index.php/fonts#Persistent_configuration)
and the [Wikipedia](https://en.wikipedia.org/wiki/ISO/IEC_8859#The_parts_of_ISO/IEC_8859) entries.

1. $ `nano /etc/vconsole.conf`
    > To set permanent setting for the console add the lines:
    > ```bash
    > KEYMAP=uk
    > FONT=ter-228n
    > FONT_MAP=8859-1
    > ```

#### Root/User Account Configuration

1. $ `passwd`
    > Set the root password.
2. $ `useradd -m -g users -G wheel -s /bin/bash $USERNAME`
    > Adds a user. Replace $USERNAME with whatever user name you which to use.
3. $ `passwd $USERNAME`
    > Sets the user's password.

#### Initial RAM Environment Configuration ([`mkinitcpio`](https://wiki.archlinux.org/index.php/mkinitcpio))

1. $ `nano /etc/mkinitcpio.conf`
    > In the file do the following:
    > * In 'MODULES: 
    >   1. Add `ext4`
    > * In 'HOOKS':
    >   1. Add `encrypt` and `lvm2` before `filesystems`
    >   2. Add `consolefont` right after `autodetect` (to avoid squinting at the crypto volume password prompt)
    >   3. Move `keyboard` right after `consolefont`
    >
    >   HOOKS should be ordered as such in the end:  
        `HOOKS=(base udev autodetect consolefont keyboard modconf block encrypt lvm2 filesystems fsck)`

2. $ `mkinitcpio -p linux`
    > Regenerates the initrd image.

#### GRUB Setup

1. $ `grub-install`
2. $ `nano /etc/default/grub`

    > At line with `GRUB_CMDLINE_LINUX` add the arguments so that it becomes 
      `GRUB_CMDLINE_LINUX="cryptdevice=/dev/sda3:luks:allow-discards"`
       
3. $ `grub-mkconfig -o /boot/grub/grub.cfg`

#### Dismount and reboot

1. $ `umount -R /mnt`
2. $ `swapoff -a`

Remove the USB stick then `reboot`.

__Note__: If you want a break this is the place to take it. Instead of rebooting just shutdown
the machine. :zzz:

## System Setup II: Hardware and Tools

Reconnect to your wifi with `sudo wifi-menu`

#### Enabling the multilib package repository

1. $ `nano /etc/pacman.conf`
    > Uncomment the multilib lines (i.e. remove the `#`):
    > ```bash
    > #[multilib]
    > #Include = /etc/pacman.d/mirrorlist
    > ```
2. $ `pacman -Syuu` 
    > Updates the cache


#### AUR package manager

From your home directory (`cd ~`):

1. $ `mkdir -p git-repos/system-builds`
2. $ `cd git-repos/system-builds`
    > You can have a different folder where to clone the repositories into. This is just mine.
3. $ `git clone https://aur.archlinux.org/package-query.git`
    > Required for [Yaourt](https://github.com/archlinuxfr/yaourt).
4. $ `git clone https://aur.archlinux.org/yaourt.git` 
    > Easy AUR package installations from the console.

    For each of the 2 packages, `cd` into their respective directories and run the following 
    command to install:

4. $ `makepkg -si`
    > e.g.: `cd package-query` then `makepkg -si`.

#### Console-Fu

1. $ `yaourt -S hstr-git`
    > This is a replacement on [steroids](https://github.com/dvorka/hstr) for the console's `ctrl+r`.
2. $ `hh --show-configuration >> ~/.bashrc`
    > Adds the config options for HSTR to your bash profile and auto-starts hh on login.
3. $ `pacman -S powertop htop`
    > Installs 2 of the most basic and useful monitoring tools in linux.
    
#### Power management

1. $ `yaourt -S laptop-mode-tools`
    > All the laptop-centric [power saving](https://wiki.archlinux.org/index.php/Laptop_Mode_Tools) 
      goodies
2. $ `sudo systemctl enable laptop-mode.service`
    > Turns on the service 
3. $ `pacman -S acpid` (Optional)
    > [Daemon](https://wiki.archlinux.org/index.php/Acpid) for delivering ACPI events.

#### Hardware

##### Fans

1. $ `yaourt -S macfanctld`
    > [macfanctld](https://github.com/MikaelStrom/macfanctld) controls the mbp fans.
2. $ `sudo systemctl enable macfanctld.service`
    > Turns on the service

##### Processor

1. $ `intel-ucode`
    > For intel's [microcode](https://wiki.archlinux.org/index.php/microcode) update. 
2. $ `grub-mkconfig -o /boot/grub/grub.cfg`
    > Need to update grub so that it loads the Microcode updates at boot
3. $ `yaourt -S msr-tools`
    > [MSR-Tools](https://01.org/msr-tools) will be used to turn off boost on the processor 
      from a script.
    
##### Sound

[Alsa](https://wiki.archlinux.org/index.php/ALSA) works without issues.

1. $ `sudo pacman -S alsa-utils alsa-plugins`
2. $ `alsamixer`
    > Make sure your current sound card is the "HDA Intel PCH" and that your master volume is up 
      and unmuted (mute=MM, unmuted=00 at the bottom of the volume bar. You can use the `M` key 
      on the keyboard to toggle mute).
3. $ `speaker-test -c 2`
    > To make sure the sound works.      

__Note__: The internal speaker might not be disabled when using the headphone jack. To solve 
this, enable "Auto-mute" in alsamixer.

##### Bluetooth

[Bluetooth](https://wiki.archlinux.org/index.php/Bluetooth) works out-of-the-box with the 
standard packages.

1. $ `sudo pacman -S bluez bluez-utils`
2. $ `modprobe btusb`
3. $ `sudo systemctl enable bluetooth.service`

##### Video

1. $ `sudo pacman -S mesa xf86-video-amdgpu vulkan-radeon lib32-mesa`
    > Installs the open source drivers for the ATI GPU along with the 32bit libs 
      and the Vulkan drivers.  
      The [`lm_sensors`](https://wiki.archlinux.org/index.php/Lm_sensors) package used for 
      temperature monitoring is a dependency for `mesa` so will be installed with it.
      Run `sensors` to see all the temperatures.

##### Webcam

The 2015 MBP has [Facetime HD](https://wiki.archlinux.org/index.php/MacBook#Facetime_HD). 
Fortunately there is a reversed-engineered driver but "PC suspension is not supported if a 
program that is keeping the camera active is running".

1. $ `yaourt -S bcwc-pcie-git`

## System Setup III: Desktop

Wayland support as of Dec 2017 on KDE is beta at best. Scaling in nice but buggy as hell displaying 
artifacts. Until that gets a lot better Xorg is the default choice as display managers go even if
HiDPI support is terrible. 

#### Display manager ([Xorg](https://wiki.archlinux.org/index.php/xorg))

1. $ `xorg, xorg-server xorg-xinit`

#### Desktop Environment ([KDE](https://wiki.archlinux.org/index.php/KDE))

##### What works

* Keyboard and the backlight
* Bluetooth? //TODO check

1. $ `sudo pacman -S plasma kde-applications sddm systemd-kcm`
2. $ `sudo systemctl enable sddm.service`
3. $ `sudo systemctl start sddm.service`

[SDDM](https://wiki.archlinux.org/index.php/SDDM)

//TODO auto-login

#### Printing/Scanning

1. $ `sudo pacman -S cups print-manager`
2. $ //TODO scanning


## System Setup IV: Applications

#### Console apps

`cmus`
`wget`

#### Desktop apps

$ `yaourt -S pamac-aur`

$ `pacman -S yakuake`


## Bugs

//TODO `kfd kfd: kgd2kfd_probe failed` at start up

//TODO `brcmfmac: brcmf_inetaddr_changed: fail to get arp ip table err:-23` after wifi connect

#### USB suspend error

You may get the following sort of error after suspend/sleep:

```bash
usb 2-4: usb_reset_and_verify_device Failed to disable LTM  
usb usb2-port4: cannot disable (err=-32)
```

The Kernel [Bug report #117811](https://bugzilla.kernel.org/show_bug.cgi?id=117811)'s thread
suggest disabling USB auto-suspend.
* If you are using [TLP](https://wiki.archlinux.org/index.php/TLP):  
  $ `nano /etc/default/tlp` and set `USB_AUTOSUSPEND=0`  
* If you are __not__ using [TLP](https://wiki.archlinux.org/index.php/TLP):  
  $ `sudo echo on | tee /sys/bus/usb/devices/*/power/control`

## References

* [Arch Linux's MacBookPro 11.x Wiki page](https://wiki.archlinux.org/index.php/MacBookPro11,x#Using_the_MacBook.27s_native_EFI_bootloader_.28recommended.29)
* [Mattias Lindberg's Encrypted volume Arch install guide](https://gist.github.com/mattiaslundberg/8620837)
* [HowToForge | How to install Arch Linux with Full Disk Encryption](https://www.howtoforge.com/tutorial/how-to-install-arch-linux-with-full-disk-encryption/)
* []()
* []()
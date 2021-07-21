# Installing Arch Linux on a LUKS Encrypted Drive using LVM booting with UEFI
This document describes my preferred way to install Arch Linux.

- **LUKS** allows full disk encryption.

- **LVM** (Logical Volume Management) is a more flexible way to set up a hard drive, as it allows partitions to be dynamically resized.

- **UEFI** is the modern replacement for Legacy BIOS.

______________________________________________________________________________

Before you begin you must prepare your installation media and make sure your BIOS is configured correctly:

## Prepare Installation Media

[Download](https://www.archlinux.org/download/) the Arch Linux ISO and create a bootable USB drive using the `dd` command:
```
#   sudo dd bs=4M if=/path_to_arch_.iso of=/dev/sd*
```
On Mac or Windows use [Ventoy](https://www.ventoy.net/).
______________________________________________________________________________

## BIOS Configuration

Hold F12 (or whatever key is used on your system) during startup to access bios. Then ...

- **Make sure UEFI is ON**. Most modern systems use UEFI, so it's generally on by default.

- **Disable Secure Boot**. If secure boot is enabled it must be turned off since Linux boot loaders don't typically have digital signatures. Note that if you intend on running a dual-boot system with Windows and Linux you won't be able to use disk encryption on the partition containing Windows, as it requires secure boot.

- **Disable Fast Startup Mode**. If you are dual booting with Windows turn off Fast Startup. This feature puts Windows into hibernation when you power off. Because some systems are still active during hibernation, booting into Linux can cause various nasty problems.
______________________________________________________________________________

## Important: Regarding Disk Node Names

All references to disk nodes in this document are shown as:

```
/dev/sd*
```

You will need to change `sd*` to the actual node name you want to use on your drive. To get this info use either of these commands:

```
fdisk -l
```
```
lsblk
```

Drive nodes might be called "sda", "sdb", etc., or they might be something completely different. On my HP Envy, for example, they are called "nvme0n1" to indicate they are interfaced via PCIe.
______________________________________________________________________________

# Installation Steps

Here we go ...

## Boot Arch from the USB Drive

Hold F12 (or whatever key is used on your system) during startup to access startup menu. Select the USB drive and boot into Arch.
______________________________________________________________________________

## Establish an Internet Connection

The most reliable way is to use a wired connection, as Arch is setup by default to connect to DHCP. However, you can usually get WiFi working by running:

```
#   iwctl
```
```
station wlan0 connect <SSID>
```

To test your connection:

```
#   ping -c 5 google.com
```
______________________________________________________________________________

## Increase Terminal Font Size

If the terminal font is too small, which can happen if you have a high res display, then increase terminus font size.

```
#   setfont ter-132n
```
______________________________________________________________________________

## Zero Hard Drive with Random Data

Optional step if you are using a hard drive with existing data. Here's how to do it using dd:

```
#   dd if=/dev/urandom of=/dev/sd* status=progress
```

Or if you're paranoid (and have a couple days to wait) you can use a multi-pass tool like shred.

```
#   shred -vfz -n 3 /dev/sd*
```

## Partition Hard Drive

First, launch **fdisk** on your desired drive node:

```
#   fdisk /dev/sd*
```

**Then** crate a GPT-Partitiontable. Therefore type only `g` and hit `ENTER`

Now run the following commands with your particular size values. Sizes can be specified in M, G, or as a percentage:

#### EFI Partition

```
n
ENTER
ENTER
+500M
t
1 (for EFI)
```

#### Boot partition

```
n
ENTER
ENTER
+500M
```

#### LVM Partition

```
n
ENTER
ENTER
ENTER (left space)
t
30 (Linux LVM)
```

Then control it with `p`(for print) and save it with `w`(for write).
______________________________________________________________________________

## Disk Encryption

Before we setup our LVM we need to encrypt the root partition we just created. I use AES 256. Note that cryptsetup splits the supplied key in half, so to use AES-256 we set the key size to 512. For more information about LUKS you can visit the Arch Wiki.

**Note:** For drives larger than 2TB use `aes-xts-plain64` instead of `aes-xts-plain`

```
#   cryptsetup luksFormat -c aes-xts-plain -y -s 512 -h sha512 /dev/sd*3
```

Now let's decrypt it so we can use it.

**Note:** I'm labeling this partition as `lvm`. We will use this label later when we create the LVM.

```
#   cryptsetup luksOpen /dev/sd*3 lvm
```
______________________________________________________________________________

# LVM Setup

## Create a Physical Volume

```
#   pvcreate /dev/mapper/lvm
```

## Create a Volume Group

**Note:** I'm labelling my volume group as `vg0`. If you use something else, make sure to replace every instance of it, not only in this section, but also in the bootloader config section much later.

```
#   vgcreate vg0 /dev/mapper/lvm
```

## Create the Logical Volumes

At minimum we need one volume, for root. Additionally we can put home on its own volume.

Also the `L` arguments below are case sensitive. The capital `L` is used when you want to specify a fixed size volume, the lowercase `l` lets you specify percentages.

```
#   lvcreate -L 50GB -n lv_root vg0
#   lvcreate -l 100%FREE -n lv_home vg0
```

Now activate the volumegroup:

```
#   modprobe dm-crypt
#   vgscan
#   vgchange -ay
```
______________________________________________________________________________

# Create the Filesystems

Note: The boot and EFI partitions are on non-LVM partitions, so use the disk node you specified when you created these partitions.

#### lv_root
```
#   mkfs.ext4 /dev/vg0/lv_root
```
#### lv_home
```
#   mkfs.ext4 /dev/vg0/lv_home
```
#### Boot Partition
```
#   mkfs.ext4 /dev/sd*2
```
#### EFI Partition
```
#   mkfs.fat -F32 /dev/sd*1
```
______________________________________________________________________________

# Mount the volumes

We need to create a couple directories while we're at it.

#### Root Volume
```
#   mount /dev/vg0/lv_root /mnt
```
#### Home Volume
```
#   mkdir /mnt/home
#   mount /dev/vg0/lv_home /mnt/home
```
#### Boot Partition
```
#   mkdir /mnt/boot
#   mount /dev/sd*2 /mnt/boot
```
______________________________________________________________________________

# Generate fstab

We now need to update the filesystem table on the new installation. Fstab contains the association between filesystems and mountpoints.

```
#   mkdir /mnt/etc
#   genfstab -Up /mnt >> /mnt/etc/fstab
```

You can verify fstab with:

```
#   cat /mnt/etc/fstab
```
______________________________________________________________________________

# Update mirrorlist

Before we download the Arch packages we should rank the mirrorlist to ensure our download speeds are as good as possible, and that the server being used is from our locale. Usually I do this with Reflector.

First, make sure the pacman databases are up-to-date:

```
#   pacman -Syy
```

Install *Reflector*:

```
#   pacman -S reflector rsync curl
```

**Note:** First make a backup copy of the mirrorlist file:

```
#   cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
```

Now generate the new mirrorlist.
**Note:** If you are in a different country change `United States` to your country.

```
#   reflector --verbose --country 'United States' -l 5 -p https --sort rate --save /etc/pacman.d/mirrorlist
```

Now refresh the package list:

```
#   pacman -Syy
```
______________________________________________________________________________

# Install Arch Linux base

First install only an Arch base with no additional packages:

```
pacstrap -i /mnt base base-devel
```
______________________________________________________________________________

# Change Root

Since we're still booted via USB, in order to configure our new system we need to change root. If we don't do that, every change we make will be applied to the USB installation.

```
#   arch-chroot /mnt
```
______________________________________________________________________________

# Install additional packages

In order to have a working system we also have to install some basic packages like the Linux kernel.

**Note:** I install `intel-ucode` to allow the Linux kernel to update the [processor microcode](https://wiki.archlinux.org/index.php/microcode). If you're on an AMD Prozessor you need to install the `amd-ucode` package instead.

```
#   pacman -S git nano vim bash-completion grub efibootmgr dosfstools os-prober mtools linux linux-headers linux-firmware lvm2 cryptsetup net-tools networkmanager network-manager-applet netctl wireless_tools wpa_supplicant dialog intel-ucode
```

Also don't forget to enable `NetworkManager` in order to have a internet connection after reboot:

**Note:** It is very important to write `NetworkManager` with a capital `N` and `M`

```
#   systemctl enable NetworkManager
```
______________________________________________________________________________

# Install and configure bootloader

```
#   nano /etc/default/grub
```

Scroll to the very top. It should look similar to this:

`GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"`

Change it to this:

`GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=/dev/sda3:vg0:allow-discards loglevel=3 quiet"`

Also uncomment:

`#GRUB_ENABLE_CRYPTODISK=y`


## Mount EFI Partition

```
#   mkdir /boot/EFI
#   mount /dev/sd*1 /boot/EFI
```

## Install Grub

Execute the following command to install the GRUB EFI application `grubx64.efi` to `/boot/EFI/grub_uefi/` and install its modules to `/boot/grub/x86_64-efi/`:

```
#   grub-install --target=x86_64-efi --efi-directory=/boot/EFI --bootloader-id=grub_uefi --recheck --debug
```

```
#   mkdir -p /boot/grub/locale

#   cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo

#   grub-mkconfig -o /boot/grub/grub.cfg
```
______________________________________________________________________________

# Update mkinitcpio

Since we're using disk encryption we need to make sure that the LUKS module gets initialized by the kernel so we can decrypt our drive prior to booting. We also need to make sure that the keyboard is available for use prior to initializing the filesystem, otherwise we will have no input device to type in our password.

Edit the following config file:

```
#   nano /etc/mkinitcpio.conf
```

Scroll down to the HOOKS section. It should look similar to this:

```
HOOKS=(base udev autodetect modconf block filesystems keyboard fsck)
```

Change it to this:

```
HOOKS=(base udev autodetect modconf block encrypt lvm2 filesystems keyboard fsck)
```

Now update the initramfs image with our changes:

```
#   mkinitcpio -p linux
```

If your computer is running PCIe storage rather than SATA add `nvme` to MODULES. NVMe is a specification for accessing SSDs attached through the PCI Express bus. The Linux kernel includes an NVMe driver, so we just need to tell the kernel to load it.

Scroll up to the MODULES section and change it to:

```
MODULES=(nvme)
```

If you did add the NVMe-Module update the initramfs image again:

```
#   mkinitcpio -p linux
```

If you're curious what modules are available as intcpio hooks, run:

```
#   ls /usr/lib/initcpio/install
```
______________________________________________________________________________

# Configurations

## Set Language and Locales

Open the locale.gen file and uncomment `en_US.UTF-8` and other needed [locales](https://wiki.archlinux.org/index.php/Locale):

```
nano /etc/locale.gen
```

**Note:** You need to uncomment `en_US.UTF-8` in every case, no matter in wich country you are.

Now save the file and generate the locale(s):

```
#   locale-gen
```

Copy your language choice to the locale.conf file (I'm using en_US.UTF-8):

```
#   echo LANG=en_US.UTF-8 > /etc/locale.conf
```

## Set timezone

Run this command and answer the questions to find your timezone:

```
#   tzselect
```

Now use the timezone you just looked up to create a symbolic link to `/etc/localtime`

**Note:** Be sure to change `America/Denver` to your timezone.

```
#   ln -sf /usr/share/zoneinfo/America/Denver /etc/localtime
```

Update the hardware clock. I use UTC:

```
#   hwclock --systohc --utc
```

## Set Hostname

This is the name of your computer. I name mine "Arch", but you can change it to whatever you want your host to be.

```
#   echo Arch > /etc/hostname
```
______________________________________________________________________________

# Set the Root password
```
#   passwd
```
______________________________________________________________________________

# Create a user Account

Make sure to replace `<username>` with your username.

```
#   useradd -m -g users -G wheel <username>
```

And set the user password:

```
#   passwd <username>
```

## Grant User Sudo Powers

Then run the following command, which will open the sudoers file:

```
#   nano /etc/sudoers
```

Find this line and remove *only* the hashtag:

```
#%wheel ALL=(ALL) ALL
```
______________________________________________________________________________
  
# Set-up swap

First create an 8GB file:

```
#   dd if=/dev/zero of=/swapfile bs=1M count=8192 status=progress
```

Then adjust the permissions:

```
#   chmod 600 /swapfile
```

Create and Activate the swap:

```
#   mkswap /swapfile
#   swapon /swapfile
```

Add the swapfile to your **fstab** to use it on Boot-up

```
#   echo '/swapfile none swap defaults 0 0' >> /etc/fstab
```
______________________________________________________________________________

# Congratulations!

You should now have a working Arch Linux installation. It doesn't have a desktop environment or any applications yet ... For that you can check out my Automatic-Arch-Installation repo, but the base installation is done.

First, exit chroot:

```
#   exit
```

Then unmount all devices and reboot

```
#   umount -a
#   reboot
```

**Note:** You can safely ignore the error messages, that some devices are busy.

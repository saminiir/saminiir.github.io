---
layout: post
title:  "Installing Arch Linux on Dell XPS 15"
date:   2016-06-24 18:00:00
categories: [linux, arch linux]
permalink: installing-arch-linux-on-dell-xps-15/
---

So after using a Fedora/Macbook for a while, I got a new work laptop. This time I opted for the Dell XPS 15 with all the bells and whistles[^dell-xps-15]. It has received great reviews and has been compared to the quality of the Macbook Pro's.

Installing a UNIX distro is always time-consuming and depending on your desires, the configuration varies a lot. This applies especially to bare-bones distros like Arch Linux: It only provides a compiled kernel and a minimal set of system components (systemd). Rest of the system layout you have to devise yourself.

The overall layout of the final Arch Linux system will be:

* UEFI for the boot-system
* GPT for the partition table
* LUKS for full-disk encryption
* LVM for managing volumes on top of LUKS
* root and home logical volumes (ext4) and a swap volume
* systemd-boot for the boot manager

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# Booting up Arch Linux

My favorite way of booting up a Linux distro is just writing the ISO image to a USB drive.

{% highlight bash %}
$ dd bs=4M if=~/Downloads/archlinux-2016.06.01-dual.iso of=/dev/sdX status=progress && sync
{% endhighlight %}

This is fast and efficient. The USB drive is ready to be booted from. 'Nuff said.

If the system does not detect the bootable USB drive, try disabling Secure Boot if you have UEFI enabled.
Extra note regarding the NVMe-based SSD drive: I had to change SATA configuration to AHCI (not RAID) from BIOS to get it visible as a block device.

Alternatively, you can check out a way of PXE booting Arch Linux from my [previous blog post]({% post_url 2015-04-03-boot-arch-linux-from-rpi %}).

# Partitioning: UEFI, GPT and ESP

First, wipe the hard disk. This is important for enhancing the security of the disk encryption, since if the disk is full of random data, info about it is harder to deduce:

{% highlight bash %}
$ cryptsetup open --type plain /dev/sdXY container --key-file /dev/random
$ dd if=/dev/zero of=/dev/mapper/container status=progress
{% endhighlight %}

Now, make sure you have booted with UEFI:
{% highlight bash %}
$ efivar -l 
{% endhighlight %}

Since we zeroed out the main disk, we need to install a partition table to it. GPT is the recommended approach with UEFI:

{% highlight bash %}
$ gdisk /dev/nmve0
{% endhighlight %}

Next, we'll set up the EFI System Partition (ESP) and the main root partition:
{% highlight bash %}
# Add a partition of size 512M and change its partition table type to EFI System (1).
# Also, add a second partition that spans the rest of the volume
$ fdisk /dev/nmve0

# Should show "esp" in Flags:
$ parted /dev/nvme0n1 print

# Initialize it as FAT32
$ mkfs.fat -F32 -nESP /dev/nvme0n1p1

# Initialize the root filesystem
$ mkfs.ext4 /dev/nvme0n1p2
{% endhighlight %}

# Disk encryption: LUKS

Several methods for achieving full disk encryption are available. I chose to use LUKS (dm-crypt), since it is pretty standard and performant in Linux. LUKS utilizes block device encryption, meaning that it operates below the filesystem and everything written to the device is guaranteed to be encrypted. LUKS also adds ease-of-use into the key management.

Utilizing full disk encryption is no panacea - you will still be vulnerable to e.g. cold boot attack[^cold-boot], bootloader malware and all other sorts of nasty things. And in the end, if your system is compromized while the disk encryption is unlocked, it is absolutely of no help at that point. I am planning a separate blog post on hardening the boot process.

Encrypt your root partition. Choose a strong passphrase (I prefer the diceware[^diceware] method). Also notice, that cryptsetup requires an uppercase 'yes' for confirmation. :-)

{% highlight bash %}

$ cryptsetup luksFormat /dev/nvme0n1p2
$ cryptsetup open --type luks /dev/nvme0n1p2 lvm

{% endhighlight %}

# Volume management: LVM

The Arch Linux wiki puts it more concisely than I ever could: 

*Logical Volume Management utilizes the kernel's device-mapper feature to provide a system of partitions independent of underlying disk layout. With LVM you abstract your storage and have "virtual partitions", making extending/shrinking easier (subject to potential filesystem limitations).*[^lvm]

In other words, it just makes your life easier. I chose the method of using the Logical Volume Manager (LVM) on top of the LUKS-encrypted partition[^lvm-on-luks]. No real reason as to why, just that it seemed to be the most simple and robust choice. The major disadvantage of it is that the LVM cannot be used to span multiple physical volumes. For a laptop, this shouldn't be that much of a problem, however.

Let's now create the Physical Volume (pv) and Logical Volumes (lv's) for the main partition layout (swap, root, home):

{% highlight bash %}
$ pvcreate /dev/mapper/lvm

$ vgcreate MyVol /dev/mapper/lvm

$ lvcreate -L 8G MyVol -n swap
$ lvcreate -L 25G MyVol -n root
$ lvcreate -l 100%FREE MyVol -n home

$ mkfs.ext4 /dev/mapper/MyVol-root
$ mkfs.ext4 /dev/mapper/MyVol-home
$ mkswap /dev/mapper

$ mount /dev/mapper/MyVol-root /mnt
$ mkdir /mnt/home
$ mount /dev/mapper/MyVol-home /mnt/home
$ swapon /dev/mapper/MyVol-swap

{% endhighlight %}

# Install Arch Linux

Now it is time to install Arch Linux to our root partition.

First, mount your boot partition that we made earlier:

{% highlight bash %}

$ mkdir /mnt/boot
$ mount /dev/nvmen1p1 /mnt/boot

{% endhighlight %}

Now let's install the Arch base system with `pacstrap`:

{% highlight bash %}

$ pacstrap /mnt base base-devel

{% endhighlight %}

Other configuration, comments inlined:

{% highlight bash %}

# Generate filesystem information
$ genfstab -U /mnt >> /mnt/etc/fstab

# Chroot the installation
$ arch-chroot /mnt /bin/bash

# Uncomment en_US.UTF-8 and generate the locale
$ vi /etc/locale.gen
$ locale-gen

# Create locale.conf
$ cat >>/etc/locale.conf
LANG=en_US.UTF-8

# Set timezone
$ ln -s /usr/share/zoneinfo/Europe/Helsinki /etc/localtime

# Sync the hardware clock
$ hwclock --systohc --utc

{% endhighlight %}

Then, configure and regenerate a new initial ramdisk environment:

{% highlight bash %}

# Change to: HOOKS="... encrypt lvm2 ... filesystems ..."
$ vi /etc/mkinitcpio.conf

# Generate initramfs
$ mkinitcpio -p linux

{% endhighlight %}

# Boot manager: systemd-boot

_systemd-boot_ is a boot manager for UEFI systems. It only operates on ESPs and EFI-configured images. The project was previously known as 'gummiboot', but was merged into systemd in May 2015[^systemd-boot].

The kernels we boot with systemd-boot have to be configured with `CONFIG_EFI_STUB` enabled. This allows the UEFI firmware to act as a bootloader and start the kernel, without needing a separate boot loader such as GRUB. Luckily, the Arch Linux kernels have already been configured so.

{% highlight bash %}

# Make sure you're booted into EFI mode and can see efivars:
$ efivar -l

# Make sure your ESP partition (described earlier) is mounted at /boot
$ mount -l | grep boot

# Install systemd-boot
$ bootctl install

{% endhighlight %}

After that, you should have a folder `/boot/loader/entries`. Add the following boot entry file there:

{% highlight bash %}

# Get the UUID of your root partition
$ blkid -s UUID -o value /dev/nvme0n1p2

$ cat >>arch-encrypted-lvm.conf
title Arch Linux Encrypted LVM
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options cryptdevice=UUID=<UUID>:MyVol root=/dev/mapper/MyVol-root quiet rw
{% endhighlight %}

The `initrd /intel-ucode.img` is the microcode for Intel processors - if you have one, install the `intel-ucode` package.

Now you can exit the chroot (`exit`) and reboot. If all went well, you should be asked to unlock the root partition's encryption and be awarded with a login shell.

In the next post, I'll describe the steps to configure a straightforward userspace setup for Arch Linux.

{% include twitter.html %}

# Sources 
[^dell-xps-15]:<http://www.dell.com/us/p/xps-15-9550-laptop/pd>
[^lvm-on-luks]:<https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS>
[^cold-boot]:<https://en.wikipedia.org/wiki/Cold_boot_attack>
[^diceware]:<https://en.wikipedia.org/wiki/Diceware>
[^lvm]:<https://wiki.archlinux.org/index.php/LVM#LVM_Building_Blocks>
[^systemd-boot]:<https://www.freedesktop.org/wiki/Software/systemd/systemd-boot/>

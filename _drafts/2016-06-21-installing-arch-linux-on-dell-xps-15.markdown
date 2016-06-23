---
layout: post
title:  "Installing Arch Linux on Dell XPS 15"
date:   2016-06-21 08:00:00
categories: linux
permalink: installing-arch-linux-on-dell-xps-15/
---

So after using a Fedora/Macbook for a while, I got a new work laptop. This time I opted for the Dell XPS 15 with all the bells and whistles[^dell-xps-15]. It has received great reviews and has been compared to the quality of the Macbook Pro's.

Installing a UNIX distro is always time-consuming and depending on your desires, the configuration varies a lot. This applies especially to bare-bones distros like Arch Linux: It only provides a compiled kernel and a minimal set of system components (systemd). Rest of system layout you have to devise yourself.

The reason I am documenting this process is that especially with laptops, there are some gotchas that are always a bit arcane. For example, the performance of suspend/hibernation varies greatly depending on the laptop's hardware, as well as other components like the webcam and so on. Thus, if this guide saves anyone (or me) even a bit of head-scratching time, I think it is worth it.

The overall layout of the final system will be:

* UEFI for the boot-system
* GPT for the partition table
* GRUB for the boot loader
* Single ext4 root partition
* LUKS for full-disk encryption
* LVM for managing volumes on top of LUKS
* X11 for the windowing system
* DWM for the window management

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

# Partitioning: UEFI, GPT and GRUB

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

{% highlight bash %}

{% endhighlight %}

# Disk encryption: LUKS

Several methods for achieving full disk encryption are available. I chose to use LUKS (dm-crypt), since it is pretty standard and performant in Linux. Furthermore, I chose the method of using the Logical Volume Manager (LVM) on top of the LUKS-encrypted partition[^lvm-on-luks]. No real reason as to why, just that it seemed to be the most simple and robust choice. The major disadvantage of it is that the LVM cannot be used to span multiple volumes. For a laptop, this shouldn't be
that much of a problem however.

# Volume management: LVM

# Suspend/hibernate

/sys/kernel/pm_state



{% include twitter.html %}

# Sources 
[^dell-xps-15]:<http://www.dell.com/us/p/xps-15-9550-laptop/pd>
[^lvm-on-luks]:<https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS>
[^1]:<https://en.wikipedia.org/wiki/Man_page>

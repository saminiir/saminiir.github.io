---
layout: post
title:  "Configuring Arch Linux on Dell XPS 15"
date:   2016-06-25 08:00:00
categories: linux
permalink: configuring-arch-linux-on-dell-xps-15/
---

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

Now we've successfully booted into Arch Linux from our encrypted root partition. Let's configure it to our liking.

# User management

{% highlight bash %}

# Add new user
$ useradd -m -G wheel saminiir

# Uncomment wheel group
$ vi /etc/sudoers

# Harden the root pasword if necessary
$ passwd 

{% endhighlight %}

# Arch User Repository

The Arch User Repository (AUR) contains community-driven packages. `yaourt` is a popular front-end for it. Enable it by:

{% highlight bash %}

$ git clone https://aur.archlinux.org/package-query.git
$ cd package-query
$ makepkg -si
$ cd ..
$ git clone https://aur.archlinux.org/yaourt.git
$ cd yaourt
$ makepkg -si
$ cd ..

{% endhighlight %}

# X11 and Window Manager: dwm

Determine your 3D card:

{% highlight bash %}

$ lspci | grep -e VGA -e 3D
01:00.0 3D controller: NVIDIA Corporation GM107M [GeForce GTX 960M] (rev a2)

# Install nvidia proprietary drivers (pulls in xorg as well)
# Be sure to select nvidia and libinput if it prompts for the repo extras
$ pacman -S nvidia

$ pacman -S xorg-server-utils xorg-xinit
$ pacman -S git openssh slock terminus-font

$ git clone https://github.com/saminiir/dotfiles.git
$ ln -s ~/dotfiles/xorg/xinitrc .xinitrc
$ ln -s ~/dotfiles/xorg/Xresources .Xresources

{% endhighlight %}

Configure `dwm` to e.g. use `xterm` as terminal and change the modifier key to Windows key (`Mod4Mask`):

{% highlight bash %}

$ git clone https://aur.archlinux.org/dwm-git.git
$ cd dwm-git && makepkg
$ vi src/dwm/config.h
$ makepkg -fi

$ startx

{% endhighlight %}

`dwm` should start.

# Graphics

Let's install bumblebee for smart workswitching for the integrated and dedicated graphics:

{% highlight bash %}

$ pacman -S bumblebee primus bbswitch

# Should show the traditional 3D-accelerated gears
$ primusrun glxgears

# Reboot. After rebooting, you should see that bbswitch has disabled the card,
# which is good for saving the battery on the laptop
$ cat /proc/acpi/bbswitch
0000:01:00.0 OFF

{% endhighlight %}

# Sound

Just install `alsa-utils`, and use `alsamixer` to unmute the master channel. Should just work.

# Power Management: powertop

`powertop` is the best. Install it.

{% highlight bash %}

pacman -S powertop

{% endhighlight %}

# Suspend/hibernate

/sys/kernel/pm_state

# Browser: Firefox

I use Firefox. With a high DPI screen, however, you should increase the pixel density. Go to `about:config` and set the `layout.css.devPixelsPerPx` to 2.

# Gaming: Steam/Wine

Because Wine uses 32-bit lbiraries, you have to enable the `multilib` repo

{% highlight bash %}

# Uncomment multilib repo
$ vi /etc/pacman.conf

$ pacman -S wine lib32-alsa-lib lib32-alsa-plugins

{% endhighlight %}

Then, get the Steam installer from `steampowered.com`. Install it with wine:

{% highlight bash %}

$ wine ~/Downloads/SteamInstaller.exe

{% endhighlight %}

{% include twitter.html %}

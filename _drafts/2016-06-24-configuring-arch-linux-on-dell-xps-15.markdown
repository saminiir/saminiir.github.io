---
layout: post
title:  "Configuring Arch Linux on Dell XPS 15"
date:   2016-06-25 08:00:00
categories: linux
permalink: configuring-arch-linux-on-dell-xps-15/
---

* X11 for the windowing system
* DWM for window management

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

# Terminal: xterm

For some reason, I've stuck on using `xterm`. Even its manual page says that it needs to be rewritten, so feel free to go find an alternative.

For the rest of us, at least the following tweaks are useful:

{% highlight bash %}

# Make bash auto-complete better
$ pacman -S bash-completion

# Install Google's `noto` font package
$ pacman -S noto-fonts

# For easy command-on-key binding
$ pacman -S xbindkeys

{% endhighlight %}

Finally, use the `.Xresources` from my dotfiles:

{% highlight bash %}

$ git clone git@github.com:saminiir/dotfiles.git
$ cd ~ && ln -s ~/dotfiles/xorg/Xresources .Xresources

{% endhighlight %}

Important things for me is that Caps Lock is mapped as a Control key, and that the Alt (or whatever is left of space bar) acts as Meta. These can be found from the `dotfiles`, but I'll highlight them here too:

{% highlight bash %}

# No caps lock - Map it as ctrl instead
$ setxkbmap -option ctrl:nocaps

# Kill X11 with ctrl+alt+backspace
$ setxkbmap -option terminate:ctrl_alt_bksp

# Change keyboard layout on right shift
setxkbmap -layout us,fi -option grp:rshift_toggle

$ grep meta~/.Xresources 
xterm*metaSendsEscape: true

{% endhighlight %}

Furthermore, I like to have my tilde character lower on the keyboard, on the right side of left-shift:

{% highlight bash %}

$ grep tilde /usr/share/X11/xkb/symbols/pc
    key <LSGT> {        [ grave, asciitilde, grave, asciitilde ] };

{% endhighlight %}

# Input: Trackpad gestures (libinput)

I chose to use `libinput` over synaptics for the touchpad driver.

{% highlight bash %}

$ pacman -S libinput xf86-input-libinput
$ libinput-list-devices
$ xinput list-props "DLL06E4:01 06CB:7A13 Touchpad"

# Enable tap-click
$ xinput set-prop "DLL06E4:01 06CB:7A13 Touchpad" "libinput Tapping Enabled" 1

# Increase pointer acceleration
$ xinput set-prop "DLL06E4:01 06CB:7A13 Touchpad" "libinput Accel Speed" 1

{% endhighlight %}

These can be found from `xutils` in my dotfiles (described above).

# Sound

Just install `alsa-utils`, and use `alsamixer` to unmute the master channel. Should just work.

# Storage: NVMe SSD

*discard* is a property that SSDs generally benefit of.

Systemd has an unit integrated that periodly (once a week or so) runs the linux-util `fstrim`. Enable it:

{% highlight bash %}

$ systemctl status fstrim
$ systemctl enable fstrim.timer

{% endhighlight %}

# Graphics: Nvidia and Bumblebee

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


# Power Management: powertop, backlight

`powertop` is the best. Install it.

{% highlight bash %}

$ pacman -S powertop

{% endhighlight %}

Run calibration as many times as you like. Eventually, powertop will start to show accurate power consumption estimates.

{% highlight bash %}

$ powertop --calibrate

{% endhighlight %}

The display's backlight is a huge power drain, and it is often convenient to have a hotkey to adjust it.

{% highlight bash %}

$ pacman -S light

{% endhighlight %}

# Suspend/hibernate

Add `resume` to Kernel parameters. 

{% highlight bash %}

$ vi /boot/lodar/entries/arch-encrypted-lvm.conf
options ... resume=/dev/mapper/MyVol-swap quiet rw

{% endhighlight %}

Add the `resume` hook to initramfs.

{% highlight bash %}

$ vi /etc/mkinitcpio.conf
HOOKS="base ... lvm2 resume ..."

{% endhighlight %}

Note that the `resume` hook should be after `lvm2`!

Then, regenerate the initramfs:

{% highlight bash %}

$ mkinitcpio -p linux

{% endhighlight %}

If suspend/hibernation does not work at first, debug it by setting `pm_test`

{% highlight bash %}

$ cat /sys/power/pm_test
none core processors platform devices freezer

$ echo freezer | sudo /sys/power/pm_test
$ systemctl suspend

{% endhighlight %}

Go through the different stages (`freezer`, `devices`..) and watch the output of `journalctl -r` for what goes wrong.

# Browser: Firefox

I use Firefox. With a high DPI screen, however, you should increase the pixel density. Go to `about:config` and set the `layout.css.devPixelsPerPx` to 2.

# Gaming: Steam/Wine

Because Wine uses 32-bit libraries, you have to enable the `multilib` repo

{% highlight bash %}

# Uncomment multilib repo
$ vi /etc/pacman.conf

$ pacman -S wine lib32-alsa-lib lib32-alsa-plugins
$ pacman -S wine-mono wine_gecko winetricks

{% endhighlight %}

Setup a 32-bit wine environment:

{% highlight bash %}
$ WINEARCH=win32 WINEPREFIX=~/win32 winecfg

# Install .NET Framework 4.0, just for laughs
$ WINEARCH=win32 WINEPREFIX=~/win32 winetricks -q msxml3 dotnet40 corefonts

{% endhighlight %}

Then, get the Steam installer from `steampowered.com`. Install it with wine:

{% highlight bash %}

$ wine ~/Downloads/SteamInstaller.exe

{% endhighlight %}

# Password Management: pass

This is kind of an extra section, since password management is pretty personal. I, however, have liked the utility `pass`, which is a simplistic bash script for linux utilities like `pwgen`.

{% highlight bash %}

$ pacman -S pass

# Or if you have an existing pass-store, e.g. in a private cloud, symlink the folder ~/.password-store to it
$ pass init

{% endhighlight %}

`pass` uses your gpg-keys for encrypting/decrypting the files by default. See my blog post on how to establish GPG.

{% include twitter.html %}

# Sources 
[^trim-on-lvm]:<https://blog.christophersmart.com/2016/05/12/trim-on-lvm-on-luks-on-ssd-revisited/>

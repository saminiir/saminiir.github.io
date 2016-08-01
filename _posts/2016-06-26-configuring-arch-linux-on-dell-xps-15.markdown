---
layout: post
title:  "Configuring Arch Linux on Dell XPS 15"
date:   2016-06-26 08:00:00
categories: linux
permalink: configuring-arch-linux-on-dell-xps-15/
---

In the [previous post]({% post_url 2016-06-24-installing-arch-linux-on-dell-xps-15 %}), we've successfully booted into Arch Linux from our encrypted root partition.

Let's now configure it to our (my) liking.

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# User management

{% highlight bash %}

# Add new user
$ useradd -m -G wheel saminiir

# Uncomment wheel group
$ grep wheel /etc/sudoers
%wheel ALL=(ALL) ALL
# %wheel ALL=(ALL) NOPASSWD: ALL

# Harden the root pasword if necessary
$ passwd 

{% endhighlight %}

# Arch User Repository

The Arch User Repository (AUR) contains community-driven packages. `yaourt` is a popular front-end for it. Enable it by:

{% highlight bash %}

$ pacman -S git
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
$ pacman -S openssh slock terminus-font

$ git clone https://github.com/saminiir/dotfiles.git
$ ln -s ~/dotfiles/xorg/xinitrc .xinitrc
$ ln -s ~/dotfiles/xorg/Xresources .Xresources

{% endhighlight %}

Configure `dwm` to e.g. use `xterm` as terminal and change the modifier key to Windows key (`Mod4Mask`):

{% highlight bash %}

$ git clone https://aur.archlinux.org/dwm-git.git
$ cd dwm-git && makepkg

$ grep "define MODKEY" src/dwm/config.h
#define MODKEY Mod4Mask

$ grep "char \*termcmd" src/dwm/config.h
static const char *termcmd[]  = { "xterm", NULL };

$ makepkg -fi

$ startx

{% endhighlight %}

`dwm` should start.

# Terminal: xterm

For some reason, I'm stuck on using `xterm`. Even its manual page says that it needs to be rewritten, so feel free to go find an alternative.

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
$ setxkbmap -layout us,fi -option grp:rshift_toggle

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

For keyboard hotkeys, add the following to `xbindkeys` configuration:

{% highlight bash %}

$ grep -A1 amixer ~/.xbindkeysrc
"/usr/bin/amixer set Master 5%+"
    XF86AudioRaiseVolume
--
"/usr/bin/amixer set Master 5%-"
    XF86AudioLowerVolume
--
"/usr/bin/amixer set Master toggle"
    XF86AudioMute

{% endhighlight %}

# Storage: NVMe SSD

Trimming is an operation SSDs benefit greatly of. However, enabling it for encrypted drives is a security risk[^ssd-trim-security]. The options are to enable trim and suffer the weakened security, or at regular intervals take maintenance on the drive.

One way of minimizing writes, which can especially wear down SSD, is to use the `noatime` or `relatime` in the drives mount options. For me, this was enabled by default:

{% highlight bash %}

$ grep relatime /etc/fstab

.. relatime ..

{% endhighlight %}

# Graphics: Nvidia and Bumblebee

Let's install bumblebee for smart switching of the integrated and dedicated graphics:

{% highlight bash %}

$ pacman -S bumblebee primus bbswitch

# Should show the traditional 3D-accelerated gears
$ primusrun glxgears

# Reboot. After rebooting, you should see that bbswitch has disabled the card,
# which is good for saving the battery on the laptop
$ cat /proc/acpi/bbswitch
0000:01:00.0 OFF

{% endhighlight %}


# Power Management

## Powertop

`powertop` is the best. Install it.

{% highlight bash %}

$ pacman -S powertop

{% endhighlight %}

Run calibration as many times as you like. Eventually, powertop will start to show accurate power consumption estimates. Additionally, run it also with `--auto-tune`:

{% highlight bash %}

$ powertop --calibrate
$ powertop --auto-tune

{% endhighlight %}

Also, add a systemd service for autotuning on startup:

{% highlight bash %}

$ cat /etc/systemd/system/powertop.service
[Unit]
Description=Powertop tunings

[Service]
Type=oneshot
ExecStart=/usr/bin/powertop --auto-tune

[Install]
WantedBy=multi-user.target

$ systemctl enable powertop.service

{% endhighlight %}

## Battery status

I use the command `acpi -b` for checking battery status. You need the package `acpi` for it.

## Backlight

The display's backlight is a huge power drain, and it is often convenient to have a hotkey to adjust it.

{% highlight bash %}

$ pacman -S light

{% endhighlight %}

Now, add commands to `xbindkeys` for manipulating the backlight:

{% highlight bash %}

$ grep -A3 light ~/.xbindkeysrc
"/usr/bin/light -U 5"
    m:0x0 + c:232
    XF86MonBrightnessDown

"/usr/bin/light -A 5"
    m:0x0 + c:233
    XF86MonBrightnessUp

{% endhighlight %}

## Laptop mode

Also, activate _laptop-mode_[^laptop-mode]:

{% highlight bash %}

$ cat /etc/sysctl.d/laptop.conf
vm.laptop_mode = 5

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

## Debugging suspend/hibernation

If suspend/hibernation does not work at first, debug it by setting `pm_test`

{% highlight bash %}

$ cat /sys/power/pm_test
none core processors platform devices freezer

$ echo freezer | sudo /sys/power/pm_test
$ systemctl suspend
$ systemctl hybrid-sleep

{% endhighlight %}

Go through the different stages (`freezer`, `devices`..) and watch the output of `journalctl -r` for what goes wrong.

## Locking screen on sleep

I want to lock the screen (`slock`) whenever the system is put to sleep.

This can be achieved by setting systemd units that have the `sleep.target` activated[^systemd-sleep-service]. See more examples from `dotfiles/systemd`:

{% highlight bash %}

$ cat /etc/systemd/system/suspend@saminiir.service
[Unit]
Description=User suspend actions
Before=sleep.target

[Service]
User=%I
Type=simple
Environment=DISPLAY=:0
ExecStart=/usr/bin/slock
ExecStartPost=/usr/bin/sleep 1

[Install]
WantedBy=sleep.target

$ systemctl enable suspend@saminiir.service

{% endhighlight %}

# Networking: netctl, Unbound

Use whatever network manager that rows your boat. I have experience in NetworkManager, but the default `netctl` in Arch Linux might be worth using too.

I've used `dnsmasq` in the past, but `unbound` seems like a cool caching DNS resolver that handles DNSSEC as well.

{% highlight bash %}

$ pacman -S expat unbound

{% endhighlight %}

See the documentation how to configure it. Most importantly your `/etc/resolv.conf` should point to `127.0.0.1` to use your own DNS program.

{% highlight bash %}

$ cat /etc/resolv.conf
# Generated by resolvconf
domain lan
nameserver 127.0.0.1

{% endhighlight %}

# Browser: Firefox

I use Firefox. With a high DPI screen, however, you should increase the pixel density. Go to `about:config` and set the `layout.css.devPixelsPerPx` to 2.

# Backups: rsnapshot

I use `rsnapshot`[^rsnapshot]. TODO.

# Password Management: pass

This is kind of an extra section, since password management is pretty personal. I, however, have liked the utility `pass`, which is a simplistic bash script for linux utilities like `pwgen`.

{% highlight bash %}

$ pacman -S pass

# Or if you have an existing pass-store, e.g. in a private cloud, symlink the folder ~/.password-store to it
$ pass init

{% endhighlight %}

`pass` uses your gpg-keys for encrypting/decrypting the files by default. See my earlier [blog post]({% post_url 2016-01-24-cryptographic-identity-gpg %}) on how to establish GPG.

Then, synchronize the `~/.password-store` between computers however you like.

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
$ WINEARCH=win32 WINEPREFIX=~/win32 winetricks -q dotnet45 corefonts

{% endhighlight %}

Then, use `winetricks to install Steam:

{% highlight bash %}

$ WINEARCH=win32 WINEPREFIX=~/win32 winetricks steam

{% endhighlight %}

You should be able to start Steam/Wine. Your mileage may vary. It is easy to have missed lib32 packages on a 64-bit system, so you might have to chase down dependencies.

{% highlight bash %}

$ WINEARCH=win32 WINEPREFIX=~/wine32 primusrun wine ~/wine32/drive_c/Program\ Files/Steam/Steam.exe

{% endhighlight %}

## Gamepad

My generic xbox pad just worked out-of-the-box. See the kernel docs for more info[^xpad].

{% highlight bash %}

$ dmesg | tail -n3
[ 3508.420268] usb 1-1: new full-speed USB device number 5 using xhci_hcd
[ 3508.609392] input: Generic X-Box pad as /devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1:1.0/input/input21
[ 3508.609619] usbcore: registered new interface driver xpad

{% endhighlight %}

You can test the pad:

{% highlight bash %}

$ pacman -S joyutils
$ jstest /dev/input/js0

{% endhighlight %}

# iPhone

I like the robustness of iPhones. I tried a Sony Z5 Compact, but the first drop broke the glass. My iPhone 5 of three years has taken a substantial amount of beating and does not have a dent. Go figure.

Install `libimobiledevice` to access the iPhone in Linux:

{% highlight bash %}

$ pacman -S libimobiledevice usbmuxd
$ systemctl start usbmuxd

{% endhighlight %}

Then, connect your iPhone via USB. Have the screen unlocked. It should be detected:

{% highlight bash %}

Jun 28 18:43:24 localhost kernel: ipheth 1-2:4.2: Apple iPhone USB Ethernet device attached
Jun 28 18:43:24 localhost kernel: usbcore: registered new interface driver ipheth
Jun 28 18:43:24 localhost kernel: ipheth 1-2:4.2 enp0s20f0u2c4i2: renamed from eth0

$ idevicepair pair

{% endhighlight %}

Then, you can backup your iPhone:

{% highlight bash %}

$ idevicebackup2 backup ~/Documents/iphone-backups/

{% endhighlight %}

{% include twitter.html %}

# Sources 
[^trim-on-lvm]:<https://blog.christophersmart.com/2016/05/12/trim-on-lvm-on-luks-on-ssd-revisited/>
[^ssd-trim-security]:<https://wiki.archlinux.org/index.php/Dm-crypt/Specialties#Discard.2FTRIM_support_for_solid_state_drives_.28SSD.29>
[^systemd-sleep-service]:<https://wiki.archlinux.org/index.php/Power_management#Suspend.2Fresume_service_files>
[^laptop-mode]:<https://www.kernel.org/doc/Documentation/laptops/laptop-mode.txt>
[^rsnapshot]:<http://rsnapshot.org/>
[^xpad]:<https://www.kernel.org/doc/Documentation/input/xpad.txt>

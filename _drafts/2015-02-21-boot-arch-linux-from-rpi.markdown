---
layout: post
title:  "Booting Arch Linux from PXE (Raspberry Pi)"
date:   2015-02-21 21:40:37
categories: meta
permalink: boot-arch-linux-from-pxe
---

So over the weekend I had the great idea of reinstalling my Linux setup, mainly to incorporate [LVM](http://en.wikipedia.org/wiki/Logical_Volume_Manager_%28Linux%29) and [LUKS](http://en.wikipedia.org/wiki/Linux_Unified_Key_Setup) to the installation from the get-go.

And as if installing and configuring a new \*NIX environment is not time consuming enough, I decided to boot the installation image from [PXE](http://en.wikipedia.org/wiki/Preboot_Execution_Environment).

PXE utilizes [TFTP](http://en.wikipedia.org/wiki/Trivial_File_Transfer_Protocol) and [DHCP](http://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) to serve the installation media over the network. Luckily, instead of installing separate packages of `{% raw %}tftp-server{%endraw%}` and `{%raw%}dhcpd{%endraw%}` to your setup, [dnsmasq](http://en.wikipedia.org/wiki/Dnsmasq) offers the whole PXE-boot functionality in one seemingly clean package.

Start with downloading the image to the raspberry and mounting it:

{% highlight bash %}
$ wget http://ftp.df.lth.se/pub/archlinux/iso/2015.02.01/archlinux-2015.02.01-dual.iso
$ mkdir -p /srv/tftp
$ mount -o loop,ro /media/passport/tftp/archlinux-2015.02.01-dual.iso /srv/tftp 
{% endhighlight %}

Install and configure dnsmasq:

{% highlight bash %}
$ sudo apt-get install dnsmasq
$ vi /etc/dnsmasq.conf
{% endhighlight %}

One gotcha is that I already have a dhcp-daemon running on my router, so I want dnsmasq in Raspberry to behave as a dhcp-proxy. The configuration should look like something along the lines of:

{% highlight bash %}
pi@raspberry:~$ grep "^[^#]" /etc/dnsmasq.conf
port=0
dhcp-range=192.168.1.0,proxy,255.255.255.0
dhcp-option-force=209,boot/syslinux/archiso.cfg
dhcp-option-force=210,/arch/
enable-tftp
tftp-root=/srv/tftp
dhcp-boot=/arch/boot/syslinux/lpxelinux.0
log-dhcp
{% endhighlight %}

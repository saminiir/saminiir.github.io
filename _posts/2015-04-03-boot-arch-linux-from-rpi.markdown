---
layout: post
title:  "Booting Arch Linux from PXE (Raspberry Pi)"
date:   2015-04-03 16:43:37
categories: meta
permalink: boot-arch-linux-from-pxe
---

So over the weekend I had the great idea of reinstalling my Linux setup, mainly to incorporate [LVM](http://en.wikipedia.org/wiki/Logical_Volume_Manager_%28Linux%29) and [LUKS](http://en.wikipedia.org/wiki/Linux_Unified_Key_Setup) to the installation from the get-go.

And as if installing and configuring a new \*NIX environment is not time consuming enough, I decided to boot the installation image from [PXE](http://en.wikipedia.org/wiki/Preboot_Execution_Environment).

PXE utilizes [TFTP](http://en.wikipedia.org/wiki/Trivial_File_Transfer_Protocol) and [DHCP](http://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) to serve the installation media over the network. Luckily, instead of installing separate packages of `{% raw %}tftp-server{%endraw%}` and `{%raw%}dhcpd{%endraw%}` in  your setup, [dnsmasq](http://en.wikipedia.org/wiki/Dnsmasq) offers the whole PXE-boot functionality in one seemingly clean package.

Start by downloading the Arch Linux image to the raspberry and mounting it:

{% highlight bash %}
$ wget http://ftp.df.lth.se/pub/archlinux/iso/2015.02.01/archlinux-2015.02.01-dual.iso
$ mkdir -p /mnt/archlinux
$ mount -o loop,ro archlinux-2015.02.01-dual.iso /mnt/archlinux
{% endhighlight %}

Install dnsmasq:

{% highlight bash %}
$ apt-get install dnsmasq
{% endhighlight %}

As the case often is, there might already be a DHCP-server running in your intranet and you might not have root access to it. In this case, you can turn `dnsmasq` to behave as a proxyDHCP, effectively only serving the PXE-specific information to the client. A typical configuration for PXE boot would look something like:

{% highlight bash %}
$ grep "^[^#]" /etc/dnsmasq.conf
port=0
dhcp-range=192.168.1.0,proxy,255.255.255.0
dhcp-option=vendor:PXEClient,6,2b
dhcp-no-override
pxe-service=x86PC, "Boot from network", /pxelinux
enable-tftp
tftp-root=/srv/archlinux
log-dhcp
$ service dnsmasq restart
{% endhighlight %}

However, the default Arch Linux image has its boot configuration elsewhere than the `pxelinux.cfg`, which PXE boot will automatically look for. This coupled with the obscure fact that `dhcp-option-force` [seems not to work when running proxyDHCP](http://www.richud.com/wiki/Network_iPXE_dnsmasq_Examples_PXE_BOOT) leaves no option but to manually configure the PXE-boot options of the image. One way of doing this is to just copy the contents of the image and modify them:

{% highlight bash %}
$ mkdir -p /srv/archlinux
$ mkdir /srv/archlinux/arch
$ cp -r /mnt/archlinux/arch/boot /srv/archlinux/
$ cp -r /mnt/archlinux/arch/x86_64 /srv/archlinux/arch/
$ cd /srv/archlinux
$ ln -s boot/syslinux/lpxelinux.0 pxelinux.0
$ mkdir pxelinux.cfg
$ ln -s ../boot/syslinux/archiso.cfg pxelinux.cfg/default
{% endhighlight %}

Finally, the root filesystem from the Arch image has to be transferred somehow. The [options](https://wiki.archlinux.org/index.php/PXE#Boot) are HTTP, NFS and NBD. I opted for setting up a NFS share:

Raspberry:
{% highlight bash %}
$ service rpcbind restart # This was needed on my raspbmc
$ apt-get install nfs-kernel-server
$ tail -n1 /etc/exports
/srv/archlinux 192.168.1.0/24(ro,no_subtree_check)
{% endhighlight %}

You need to also point the PXE/NFS -mount to that folder:
{% highlight bash %}
$ grep -A10 arch64_nfs /srv/archlinux/boot/syslinux/archiso_pxe64.cfg
LABEL arch64_nfs
TEXT HELP
Boot the Arch Linux (x86_64) live medium (Using NFS).
It allows you to install Arch Linux or perform system maintenance.
ENDTEXT
MENU LABEL Boot Arch Linux (x86_64) (NFS)
LINUX boot/x86_64/vmlinuz
INITRD boot/intel_ucode.img,boot/x86_64/archiso.img
APPEND archisobasedir=arch archiso_nfs_srv=${pxeserver}:/srv/archlinux
SYSAPPEND 3

{% endhighlight %}

Now if you boot over PXE, you should eventually be awarded with a 'Welcome to Arch Linux!' message. Alternatively, you can [debug/test the PXE boot with QEMU.]({% post_url 2015-04-03-debugging-pxe-boot %})

---
layout: post
title:  "Debugging PXE boot with QEMU"
date:   2015-04-03 16:44:37
categories: meta
permalink: debugging-pxe-boot
---

In my [earlier post]({% post_url 2015-04-03-boot-arch-linux-from-rpi %}), I defined the steps for setting up a PXE boot environment.

However, you might run into configuration problems and booting an actual machine is time-consuming for testing. My situation, for example, was that I could not get the Raspberry serving the PXE-protocol to respond to legitimate requests. I needed a better environment to debug the problem than booting my desktop for every iteration. 

And here steps in QEMU. It's a fine piece of computer science originally by Fabrice Bellard, enabling super-fast virtualization of machines.  

The steps I did to procure a PXE-debugging environment in Arch Linux:

Install qemu, brctl:
{% highlight bash %}
$ pacman -S qemu
$ pacman -S brctl
{% endhighlight %}

Create a test QEMU image:
{% highlight bash %}
$ qemu-img create -f vmdk testpxe.vmdk 10  
{% endhighlight %}

Configure bridged networking, to be used for the VM:
{% highlight bash %}
$ ip tuntap add dev virttap mode tap user $USER
$ brctl addbr virtbr
$ brctl addif virtbr enp2s0 # your LAN interface
$ brctl addif virtbr virttap
$ ip link set virtbr up
$ ip link set enp2s0 up
$ ip link set virttap up
{% endhighlight %}

Finally, launch the PXE-booting VM:
{% highlight bash %}
$ /usr/bin/qemu-system-x86_64 -m 1024 -hda testpxe.vmdk \
                              -boot n \
                              -option-rom /usr/share/qemu/pxe-rtl8139.rom \
                              -net nic \
                              -net tap,ifname=virttap,script=no,downscript=no
{% endhighlight %}

It should automatically try to boot over LAN (``-boot n``).

By the way, my original problem was that my Raspberry's firewall did not accept UDP packets. D'oh!
{% highlight bash %}
iptables -I INPUT -i $IFACE -p udp --dport 67:68 --sport 67:68 -j ACCEPT
{% endhighlight %}

Enables DHCP and the PXE-boot requests were allowed through.

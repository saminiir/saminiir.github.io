---
layout: post
title:  "Let's code a TCP/IP stack, 1: Ethernet & ARP"
date:   2016-02-26 07:00:00
categories: networking
permalink: lets-code-tcp-ip-stack-1-ethernet-arp
---

Writing your own TCP/IP stack may sound like a daunting task. Indeed, TCP has accumulated many specifications over its lifetime of more than thirty years. The core specification, however, is seemingly compact[^tcp-roadmap] - the important parts being TCP header parsing, the state machine, congestion control and retransmission timeout computation.

The most common layer 2 and layer 3 protocols, Ethernet and IP respectively, pale in comparison to TCP's complexity. In this blog series, we will implement a minimal userspace TCP/IP stack for Linux.

The purpose of these posts and the resulting software is purely educational - to learn network and system programming at a deeper level.

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# TUN/TAP devices

To intercept low-level network traffic from the Linux kernel, we will use a Linux TAP device. In short, a TUN/TAP device is often used by networking userspace applications to manipulate L2/L3 traffic. A popular example is [tunneling]({% post_url 2016-02-20-openvpn-puts-packets-inside-your-packets %}#tunneling), where packets are wrapped inside the payload of another packet.

The advantage of TUN/TAP devices is that they're easy to set up in a userspace program and they are already being used in multitude of programs, such as [OpenVPN]({% post_url 2016-02-20-openvpn-puts-packets-inside-your-packets %}).

As we want to build the networking stack from the layer 2 up, we need a TAP device. We instantiate it like so:

{% highlight c %}
/*
 * Taken from Kernel Documentation/networking/tuntap.txt
 */
int tun_alloc(char *dev)
{
    struct ifreq ifr;
    int fd, err;

    if( (fd = open("/dev/net/tap", O_RDWR)) < 0 ) {
        print_error("Cannot open TUN/TAP dev");
        exit(1);
    }

    CLEAR(ifr);

    /* Flags: IFF_TUN   - TUN device (no Ethernet headers)
     *        IFF_TAP   - TAP device
     *
     *        IFF_NO_PI - Do not provide packet information
     */
    ifr.ifr_flags = IFF_TAP | IFF_NO_PI;
    if( *dev ) {
        strncpy(ifr.ifr_name, dev, IFNAMSIZ);
    }

    if( (err = ioctl(fd, TUNSETIFF, (void *) &ifr)) < 0 ){
        print_error("ERR: Could not ioctl tun: %s\n", strerror(errno));
        close(fd);
        return err;
    }

    strcpy(dev, ifr.ifr_name);
    return fd;
}
{% endhighlight %}

After this, the file descriptor `tun_fd` can be used to read and write data to the virtual device's ethernet buffer.

The flag `IFF_NO_PI` is crucial here, otherwise we end up with unnecessary packet information prepended to the Ethernet frame. You can actually take a look at the kernel's [source code](https://github.com/torvalds/linux/blob/v4.4/drivers/net/tun.c#L1306) of the tun-device driver and verify this yourself. 

# Ethernet Frame Format

# Address Resolution Protocol


# Sources
[^tcp-roadmap]:<https://tools.ietf.org/html/rfc7414>
[^tunneling]:<{% post_url 2015-04-03-boot-arch-linux-from-rpi %}>

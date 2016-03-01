---
layout: post
title:  "Let's code a TCP/IP stack, 1: Ethernet & ARP"
date:   2016-02-26 07:00:00
categories: networking
permalink: lets-code-tcp-ip-stack-1-ethernet-arp
---

Writing your own TCP/IP stack may seem like a daunting task. Indeed, TCP has accumulated many specifications over its lifetime of more than thirty years. The core specification, however, is seemingly compact[^tcp-roadmap] - the important parts being TCP header parsing, the state machine, congestion control and retransmission timeout computation.

The most common layer 2 and layer 3 protocols, Ethernet and IP respectively, pale in comparison to TCP's complexity. In this blog series, we will implement a minimal userspace TCP/IP stack for Linux. 

The purpose of these posts and the resulting software is purely educational - to learn network and system programming at a deeper level.

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# TUN/TAP devices

To intercept low-level network traffic from the Linux kernel, we will use a Linux TAP device. In short, a TUN/TAP device is often used by networking userspace applications to manipulate L2/L3 traffic. A popular example is [tunneling]({% post_url 2016-02-20-openvpn-puts-packets-inside-your-packets %}#tunneling), where a packet is wrapped inside the payload of another packet.

The advantage of TUN/TAP devices is that they're easy to set up in a userspace program and they are already being used in a multitude of programs, such as [OpenVPN]({% post_url 2016-02-20-openvpn-puts-packets-inside-your-packets %}).

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

After this, the returned file descriptor `fd` can be used to read and write data to the virtual device's ethernet buffer.

The flag `IFF_NO_PI` is crucial here, otherwise we end up with unnecessary packet information prepended to the Ethernet frame. You can actually take a look at the kernel's [source code](https://github.com/torvalds/linux/blob/v4.4/drivers/net/tun.c#L1306) of the tun-device driver and verify this yourself. 

# Ethernet Frame Format

The multitude of different Ethernet networking technologies are the backbone of connecting computers in _Local Area Networks_ (LANs). As with all physical technology, the Ethernet standard has greatly evolved from its first version[^ethernet], published by Digital Equipment Corporation, Intel and Xerox in 1980. 

The first version of Ethernet was slow in today's standards - about 10Mb/s and it utilized half-duplex communication, meaning that you either sent or received data, but not at the same time. This is why a _Media Access Control_ (MAC) protocol had to be incorporated to organize the data flow. Even to this day, _Carrier Sense, Multiple Access with Collision Detection_ (CSMA/CD) is required as the MAC method if running an Ethernet interface in half-duplex mode. 

The invention of the _100BASE-T_ Ethernet standard used twisted-pair wiring to enable full-duplex communication and higher throughput speeds. Additionally, the simultaneous increase in popularity of Ethernet switches made CSMA/CD largely obsolete. 

The different Ethernet standards are maintained by the IEEE 802.3[^ieee-802-3] working group.

Next, we'll take a look at the Ethernet Frame header. It can be declared as a C struct followingly:

{% highlight c %}
#include <linux/if_ether.h>

struct eth_hdr
{
    unsigned char dst_mac[6];
    unsigned char src_mac[6];
    unsigned short ethertype;
    unsigned char* payload;
};
{% endhighlight %}

The `dst_mac` and `src_mac` are pretty self-explanatory fields. They contain the MAC addresses of the communicating parties.

The overloaded field, `ethertype`, is a 2-octet field, that depending on its value, either indicates the length or the type of the payload. Specifically, if the field's value is greater or equal to 1536, the field contains the type of the payload (e.g. IPv4, ARP). If the value is less than that, it contains the length of the payload.

After the type field, there is a possibility of several different _tags_ for the Ethernet frame. These tags can be used to describe the _Virtual LAN_ (VLAN) or the _Quality of Service_ (QoS) type of the frame. Ethernet frame tags are excluded from our implementation, so the corresponding field also does not show up in our protocol declaration.

The field `payload` contains a pointer to the Ethernet frame's payload. In our case, this will contain an ARP or IPv4 packet. If the payload length is smaller than the minimum required 48 bytes (without tags), pad bytes are appended to the end of the payload to meet the requirement.

We also include the `if_ether.h` Linux header to provide a mapping between ethertypes and their hexadecimal values.

Lastly, the Ethernet Frame Format also includes the _Frame Check Sequence_ field in the end, which is used with _Cyclic Redundancy Check_ (CRC) to check the integrity of the frame.

# Address Resolution Protocol


# Sources
[^tcp-roadmap]:<https://tools.ietf.org/html/rfc7414>
[^ethernet]:<http://ethernethistory.typepad.com/papers/EthernetSpec.pdf>
[^ieee-802-3]:<https://en.wikipedia.org/wiki/IEEE_802.3>

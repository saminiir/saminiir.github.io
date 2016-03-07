---
layout: post
title:  "Let's code a TCP/IP stack, 1: Ethernet & ARP"
date:   2016-03-06 10:00:00
categories: [tcp/ip, c programming, networking, tutorial, linux]
permalink: lets-code-tcp-ip-stack-1-ethernet-arp
description: "Writing your own TCP/IP stack may seem like a daunting task. Indeed, TCP has accumulated many specifications over its lifetime of more than thirty years. The core specification, however, is seemingly compact[^tcp-roadmap] - the important parts being TCP header parsing, the state machine, congestion control and retransmission timeout computation."
---

Writing your own TCP/IP stack may seem like a daunting task. Indeed, TCP has accumulated many specifications over its lifetime of more than thirty years. The core specification, however, is seemingly compact[^tcp-roadmap] - the important parts being TCP header parsing, the state machine, congestion control and retransmission timeout computation.

The most common layer 2 and layer 3 protocols, Ethernet and IP respectively, pale in comparison to TCP's complexity. In this blog series, we will implement a minimal userspace TCP/IP stack for Linux. 

The purpose of these posts and the resulting software is purely educational - to learn network and system programming at a deeper level.

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# TUN/TAP devices

To intercept low-level network traffic from the Linux kernel, we will use a Linux TAP device. In short, a TUN/TAP device is often used by networking userspace applications to manipulate L3/L2 traffic, respectively. A popular example is [tunneling]({% post_url 2016-02-20-openvpn-puts-packets-inside-your-packets %}#tunneling), where a packet is wrapped inside the payload of another packet.

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

After this, the returned file descriptor `fd` can be used to `read` and `write` data to the virtual device's ethernet buffer.

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
    unsigned char dmac[6];
    unsigned char smac[6];
    uint16_t ethertype;
    unsigned char payload[];
} __attribute__((packed));
{% endhighlight %}

The `dmac` and `smac` are pretty self-explanatory fields. They contain the MAC addresses of the communicating parties (destination and source, respectively).

The overloaded field, `ethertype`, is a 2-octet field, that depending on its value, either indicates the length or the type of the payload. Specifically, if the field's value is greater or equal to 1536, the field contains the type of the payload (e.g. IPv4, ARP). If the value is less than that, it contains the length of the payload.

After the type field, there is a possibility of several different _tags_ for the Ethernet frame. These tags can be used to describe the _Virtual LAN_ (VLAN) or the _Quality of Service_ (QoS) type of the frame. Ethernet frame tags are excluded from our implementation, so the corresponding field also does not show up in our protocol declaration.

The field `payload` contains a pointer to the Ethernet frame's payload. In our case, this will contain an ARP or IPv4 packet. If the payload length is smaller than the minimum required 48 bytes (without tags), pad bytes are appended to the end of the payload to meet the requirement.

We also include the `if_ether.h` Linux header to provide a mapping between ethertypes and their hexadecimal values.

Lastly, the Ethernet Frame Format also includes the _Frame Check Sequence_ field in the end, which is used with _Cyclic Redundancy Check_ (CRC) to check the integrity of the frame. We will omit the handling of this field in our implementation.

# Ethernet Frame Parsing

The attribute _packed_ in a struct's declaration is an implementation detail - It is used to instruct the GNU C compiler not to optimize the struct memory layout for data alignment with padding bytes[^gnu-c-packed]. The use of this attribute stems purely out of the way we are "parsing" the protocol buffer, which is just a type cast over the data buffer with the proper protocol struct:

{% highlight c %}

struct eth_hdr *hdr = (struct eth_hdr *) buf;

{% endhighlight %}

A portable, albeit slightly more laborious approach, would be to serialize the protocol data manually. This way, the compiler is free to add padding bytes to conform better to different processor's data alignment requirements.

The overall scenario for parsing and handling incoming Ethernet frames is straightforward:

{% highlight c %}
if (tun_read(buf, BUFLEN) < 0) {
    print_error("ERR: Read from tun_fd: %s\n", strerror(errno));
}

struct eth_hdr *hdr = init_eth_hdr(buf);

handle_frame(&netdev, hdr);
{% endhighlight %} 

The `handle_frame` function just looks at the `ethertype` field of the Ethernet header, and decides its next action based upon the value.

# Address Resolution Protocol

The _Address Resolution Protocol_ (ARP) is used for dynamically mapping a 48-bit Ethernet address (MAC address) to a protocol address (e.g. IPv4 address). The key here is that with ARP, multitude of different L3 protocols can be used: Not just IPv4, but other protocols like CHAOS, which declares 16-bit protocol addresses.

The usual case is that you know the IP address of some service in your LAN, but to establish actual communications, also the hardware address (MAC) needs to be known. Hence, ARP is used to broadcast and query the network, asking the owner of the IP address to report its hardware address.

The ARP packet format is relatively straightforward:

{% highlight c %}

struct arp_hdr
{
    uint16_t hwtype;
    uint16_t protype;
    unsigned char hwsize;
    unsigned char prosize;
    uint16_t opcode;
    unsigned char data[];
} __attribute__((packed));


{% endhighlight %}

The ARP header (`arp_hdr`) contains the 2-octet `hwtype`, which determines the link layer type used. This is Ethernet in our case, and the actual value is `0x0001`.

The 2-octet `protype` field indicates the protocol type. In our case, this is IPv4, which is communicated with the value `0x0800`.

The `hwsize` and `prosize` fields are both 1-octet in size, and they contain the sizes of the hardware and protocol fields, respectively. In our case, these would be 6 bytes for MAC addresses, and 4 bytes for IP addresses.

The 2-octet field `opcode` declares the type of the ARP message. It can be ARP request (1), ARP reply (2), RARP request (3) or RARP reply (4).

The `data` field contains the actual payload of the ARP message, and in our case, this will contain IPv4 specific information:

{% highlight c %}

struct arp_ipv4
{
    unsigned char smac[6];
    uint32_t sip;
    unsigned char dmac[6];
    uint32_t dip;
} __attribute__((packed));

{% endhighlight %}

The fields are pretty self explanatory. `smac` and `dmac` contain the 6-byte MAC addresses of the sender and receiver, respectively. `sip` and `dip` contain the sender's and receiver's IP addresses, respectively.

# Address Resolution Algorithm

The [original specification](https://tools.ietf.org/html/rfc826) depicts this simple algorithm for address resolution:

{% highlight bash %}

?Do I have the hardware type in ar$hrd?
Yes: (almost definitely)
  [optionally check the hardware length ar$hln]
  ?Do I speak the protocol in ar$pro?
  Yes:
    [optionally check the protocol length ar$pln]
    Merge_flag := false
    If the pair <protocol type, sender protocol address> is
        already in my translation table, update the sender
        hardware address field of the entry with the new
        information in the packet and set Merge_flag to true.
    ?Am I the target protocol address?
    Yes:
      If Merge_flag is false, add the triplet <protocol type,
          sender protocol address, sender hardware address> to
          the translation table.
      ?Is the opcode ares_op$REQUEST?  (NOW look at the opcode!!)
      Yes:
        Swap hardware and protocol fields, putting the local
            hardware and protocol addresses in the sender fields.
        Set the ar$op field to ares_op$REPLY
        Send the packet to the (new) target hardware address on
            the same hardware on which the request was received.
{% endhighlight %}

Namely, the `translation table` is used to store the results of ARP, so that hosts can just look up whether they already have the entry in their cache. This avoids spamming the network for redundant ARP requests.

The algorithm is implemented in [arp.c](https://github.com/saminiir/level-ip/blob/e9ceb08f01a5499b85f03e2d615309c655b97e8f/src/arp.c#L53).

Finally, the ultimate test for an ARP implementation is to see whether it replies to ARP requests correctly:

{% highlight bash %}
[saminiir@localhost lvl-ip]$ arping -I tap0 10.0.0.4
ARPING 10.0.0.4 from 192.168.1.32 tap0
Unicast reply from 10.0.0.4 [00:0C:29:6D:50:25]  3.170ms
Unicast reply from 10.0.0.4 [00:0C:29:6D:50:25]  13.309ms

[saminiir@localhost lvl-ip]$ arp
Address                  HWtype  HWaddress           Flags Mask            Iface
10.0.0.4                 ether   00:0c:29:6d:50:25   C                     tap0
{% endhighlight %}

The kernel's networking stack recognized the ARP reply from our custom networking stack, and consequently populated its ARP cache with the entry of our virtual network device. Success!

# Conclusion

The minimal implementation of Ethernet Frame handling and ARP is relatively easy and can be done in a few lines of code. On the contrary, the reward-factor is quite high, since you get to populate a Linux host's ARP cache with your own make-belief Ethernet device! 

The source code for the project can be found at [GitHub](https://github.com/saminiir/level-ip).

In the next post, we'll continue the implementation with ICMP echo & reply (ping) and IPv4 packet parsing.

{% include twitter.html %}

_Kudos to Xiaochen Wang, whose similar implementation proved invaluable for me in getting up to speed with C network programming and protocol handling. I find his [source code](https://github.com/chobits/tapip)[^tapip] easy to understand and some of my design choices were straight-out copied from his implementation._

# Sources
[^tcp-roadmap]:<https://tools.ietf.org/html/rfc7414>
[^ethernet]:<http://ethernethistory.typepad.com/papers/EthernetSpec.pdf>
[^ieee-802-3]:<https://en.wikipedia.org/wiki/IEEE_802.3>
[^gnu-c-packed]:<https://gcc.gnu.org/onlinedocs/gcc/Common-Type-Attributes.html#Common-Type-Attributes>
[^tapip]:<https://github.com/chobits/tapip>

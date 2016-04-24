---
layout: post
title:  "Let's code a TCP/IP stack, 2: IPv4 & ICMPv4"
date:   2016-04-24 09:00:00
categories: [tcp/ip, tutorial, c programming, ip, icmp, networking, linux]
permalink: lets-code-tcp-ip-stack-2-ipv4-icmpv4
description: "This time in our tutorial userspace TCP/IP stack we will implement a minimum viable IP layer and test it with ICMP echo requests. We will take a look at the headers of IPv4 and ICMPv4 and describe how to check them for integrity. Some features, such as IP fragmentation, are left as an exercise for the reader."
---

This time in our userspace TCP/IP stack we will implement a minimum viable IP layer and test it with ICMP echo requests (also known as _pings_). 

We will take a look at the formats of IPv4 and ICMPv4 and describe how to check them for integrity. Some features, such as IP fragmentation, are left as an exercise for the reader.

For our networking stack IPv4 was chosen over IPv6 since it is still the default network protocol for the Internet. However, this is changing fast[^ipv6-adoption] and our networking stack can be extended with IPv6 in the future.

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# Internet Protocol version 4

The next layer (L3)[^osi-model] in our implementation, after Ethernet frames, handles the delivery of data to a destination. Namely, the _Internet Protocol_ (IP) was invented to provide a base for transport protocols such as TCP and UDP. It is connectionless, meaning that unlike TCP, all of the datagrams are handled independently of each other in the networking stack. This also means that IP datagrams may arrive out-of-order.[^stevens-tcpip]

Furthermore, IP does not guarantee successful delivery. This is a conscious choice taken by the protocol designers, since IP is meant to provide a base for protocols that likewise do not guarantee delivery. UDP is one such protocol.

If reliability between the communicating parties is required, a protocol such as TCP is used on top of IP. In that case, the higher level protocol is responsible for detecting missing data and making sure all of it is delivered.

## Header Format

The IPv4 header is typically 20 octets in length. The header can contain trailing options, but they are omitted from our implementation. The meaning of the fields is relatively straightforward and can be described as a C struct:

{% highlight c %}
struct iphdr {
    uint8_t version : 4;
    uint8_t ihl : 4;
    uint8_t tos;
    uint16_t len;
    uint16_t id;
    uint16_t flags : 3;
    uint16_t frag_offset : 13;
    uint8_t ttl;
    uint8_t proto;
    uint16_t csum;
    uint32_t saddr;
    uint32_t daddr;
} __attribute__((packed));
{% endhighlight %}

The 4-bit `version` field indicates the format of the Internet header. In our case, the value will be 4 for IPv4.

The _Internet Header Length_ field `ihl` is likewise 4 bits in length and indicates the number of 32-bit _words_ in the IP header. Because the field is 4 bits in size, it can only hold a maximum value of 15. Thus the maximum length of an IP header is 60 octets (15 times 32 divided by eight).

The _type of service_ field `tos` originates from the first IP specification[^ipv4-spec]. It has been divided into smaller fields in later specifications, but for simplicity's sake, we will treat the field as defined in the original specification. The field communicates the quality of service intended for the IP datagram.  

The _total length_ field `len`  communicates the length of the whole IP datagram. As it is a 16-bit field, the maximum length is then 65535 bytes. Large IP datagrams are subject to fragmentation, in which they are split into smaller datagrams in order to satisfy the _Maximum Transmission Unit_ (MTU) of different communication interfaces.

The `id` field is used to index the datagram and is ultimately used for reassembly of fragmented IP datagrams. The field's value is simply a counter that is incremented by the sending party. In turn, the receiving side knows how to order the incoming fragments.

The `flags` field defines various control flags of the datagram. In specific, the sender can specify whether the datagram is allowed to be fragmented, whether it is the last fragment or that there's more fragments incoming.

The _fragment offset_ field, `frag_offset`, indicates the position of the fragment in a datagram. Naturally, the first datagram has this index set to 0.

The `ttl` or _time to live_ is a common attribute that is used to count down the datagram's lifetime. It is usually set to 64 by the original sender, and every receiver decrements this counter by one. When it hits zero, the datagram is to be discarded and possibly an ICMP message is replied back to indicate an error.

The `proto` field provides the datagram an inherent ability to carry other protocols in its payload. The field usually contains values such as 16 (UDP) or 6 (TCP), and is simply used to communicate the type of the actual data to the receiver. 

The _header checksum_ field, `csum`, is used to verify the integrity of the IP header. The algorithm for it is relatively simple, and will be explained further down in this tutorial.

Finally, the `saddr` and `daddr` fields indicate the source and destination addresses of the datagram, respectively. Even though the fields are 32-bit in length and thus provide a pool of approximately 4.5 billion addresses, the address range is going to be exhausted in the near-future[^ipv4-exhaustion]. The IPv6 protocol extends this length to 128 bits and as a result, future-proofs the address range of the Internet Protocol, perhaps permanently.

## Internet Checksum

The Internet checksum field is used to check the integrity of an IP datagram. Calculating the checksum is relatively simple and is defined in the original specification[^ipv4-spec]:

  _The checksum field is the 16 bit one’s complement of the one’s complement sum of all 16 bit words in the header.  For purposes of computing the checksum, the value of the checksum field is zero._

The actual[^internet-checksum-spec] code for the algorithm is as follows:  
{% highlight c %}
uint16_t checksum(void *addr, int count)
{
    /* Compute Internet Checksum for "count" bytes
     *         beginning at location "addr".
     * Taken from https://tools.ietf.org/html/rfc1071
     */

    register uint32_t sum = 0;
    uint16_t * ptr = addr;

    while( count > 1 )  {
        /*  This is the inner loop */
        sum += * ptr++;
        count -= 2;
    }

    /*  Add left-over byte, if any */
    if( count > 0 )
        sum += * (uint8_t *) addr;

    /*  Fold 32-bit sum to 16 bits */
    while (sum>>16)
        sum = (sum & 0xffff) + (sum >> 16);

    return ~sum;
}
{% endhighlight %}

Take the example IP header `45 00 00 54 41 e0 40 00 40 01 00 00 0a 00 00 04 0a 00 00 05`:

1. Adding the fields together yields the two's complement sum `01 1b 3e`.
2. Then, to convert it to one's complement, the carry-over bits are added to the first 16-bits: `1b 3e` + `01` = `1b 3f`.
3. Finally, the one's complement of the sum is taken, resulting to the checksum value `e4c0`.

The IP header becomes `45 00 00 54 41 e0 40 00 40 01 e4 c0 0a 00 00 04 0a 00 00 05`.

The checksum can be verified by applying the algorithm again and if the result is 0, the data is most likely good. 

# Internet Control Message Protocol version 4

As the Internet Protocol lacks mechanisms for reliability, some way of informing communicating parties of possible error scenarios is required. As a result, the _Internet Control Message Protocol_ (ICMP)[^icmpv4-spec] is used for diagnostic measures in the network. An example of this is the case where a gateway is not reachable - the network stack that recognizes this sends an ICMP "Gateway Unreachable" message back to the origin.

## Header Format

The ICMP header resides in the payload of the corresponding IP packet. The structure of the ICMPv4 header is as follows:

{% highlight c %}
struct icmp_v4 {
    uint8_t type;
    uint8_t code;
    uint16_t csum;
    uint8_t data[];
} __attribute__((packed));
{% endhighlight %}

Here, the `type` field communicates the purpose of the message. 42 different[^iana-icmp-types] values are reserved for the type field, but only about 8 are commonly used. In our implementation, the types 0 (Echo Reply), 3 (Destination Unreachable) and 8 (Echo request) are used.

The `code` field further describes the meaning of the message. For example, when the type is 3 (Destination Unreachable), the code-field implies the reason. A common error is when a packet cannot be routed to a network: the originating host then most likely receives an ICMP message with the type 3 and code 0 (Net Unreachable).

The `csum` field is the same checksum field as in the IPv4 header, and the same algorithm can be used to calculate it. In ICMPv4 however, the checksum is _end-to-end_, meaning that also the payload is included when calculating the checksum.

## Messages and their processing

The actual ICMP payload consists of query/informational messages and error messages. First, we'll look at the Echo Request/Reply messages, commonly referred to as "pinging" in networking:

{% highlight c %}
struct icmp_v4_echo {
    uint16_t id;
    uint16_t seq;
    uint8_t data[];
} __attribute__((packed));
{% endhighlight %}

The message format is compact. The field `id` is set by the sending host to determine to which process the echo reply is intended. For example, the process id can be set in to this field.

The field `seq` is the sequence number of the echo and it is simply a number starting from zero and incremented by one whenever a new echo request is formed. This is used to detect if echo messages disappear or are reordered while in transit.

The `data` field is optional, but often contains information like the timestamp of the echo. This can then be used to estimate the round-trip time between hosts.

Perhaps the most common ICMPv4 error message, _Destination Unreachable_, has the following format:

{% highlight c %}
struct icmp_v4_dst_unreachable {
    uint8_t unused;
    uint8_t len;
    uint16_t var;
    uint8_t data[];
} __attribute__((packed));
{% endhighlight %}

The first octet is unused. Then, the `len` field indicates the length of the original datagram, in 4-octet units for IPv4. The value of the 2-octet field `var` depends on the ICMP code.

Finally, as much as possible of the original IP packet that caused the Destination Unreachable state is placed into the `data` field.

# Testing the implementation

From a shell, we can verify that our userspace networking stack responds to ICMP echo requests:

{% highlight bash %}
[saminiir@localhost ~]$ ping -c3 10.0.0.4
PING 10.0.0.4 (10.0.0.4) 56(84) bytes of data.
64 bytes from 10.0.0.4: icmp_seq=1 ttl=64 time=0.191 ms
64 bytes from 10.0.0.4: icmp_seq=2 ttl=64 time=0.200 ms
64 bytes from 10.0.0.4: icmp_seq=3 ttl=64 time=0.150 ms

--- 10.0.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.150/0.180/0.200/0.024 ms
{% endhighlight %}

# Conclusion

A minimum viable networking stack that handles Ethernet frames, ARP and IP can be created relatively easily. However, the original specifications have been extended with many new ones. In this post, we skimmed over IP features such as options, fragmentation and the header DCN and DS fields.

Furthermore, IPv6 is crucial for the future of the Internet. It is not yet ubiquitous but being a newer protocol than IPv4, it definitely is something that should be implemented in our networking stack.

The source code for this blog post can be found at [GitHub](https://github.com/saminiir/level-ip).

In the next blog post we will advance to the transport layer (L4) and start implementing the notorious _Transmission Control Protocol_ (TCP). Namely, TCP is a connection-oriented protocol and ensures reliability between both communicating sides. These aspects obviously bring about more complexity, and being an old protocol, TCP has its dark corners.

{% include twitter.html %}

# Sources
[^tcp-roadmap]:<https://tools.ietf.org/html/rfc7414>
[^ipv4-spec]:<http://tools.ietf.org/html/rfc791>
[^icmpv4-spec]:<https://www.ietf.org/rfc/rfc792.txt>
[^internet-checksum-spec]:<https://tools.ietf.org/html/rfc1071>
[^iana-icmp-types]:<http://www.iana.org/assignments/icmp-parameters/icmp-parameters.xhtml>
[^stevens-tcpip]:<https://en.wikipedia.org/wiki/TCP/IP_Illustrated#Volume_1:_The_Protocols>
[^osi-model]:<https://en.wikipedia.org/wiki/OSI_model>
[^ipv6-adoption]:<https://en.wikipedia.org/wiki/IPv6_deployment>
[^ipv4-exhaustion]:<https://en.wikipedia.org/wiki/IPv4_address_exhaustion>

---
layout: post
title:  "Let's code a TCP/IP stack, 2: IPv4 & ICMPv4"
date:   2016-03-12 10:00:00
categories: [tcp/ip, tutorial, c programming, ip, icmp, networking, linux]
permalink: lets-code-tcp-ip-stack-2-ipv4-icmpv4
description: "TODO: Fill"
---

Test

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# Internet Protocol version 4

## Header Format

The IPv4 header is somewhat lengthy, 20 octets in total. The meaning of the fields is relatively straightforward, however, depicted as a C struct:

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

The 4-bit `version` field indicates the format of the internet header. In our case, this will be the value 4 for IPv4.

The _Internet Header Length_ ,`ihl`,  field is likewise 4 bits in length and indicates the number of 32-bit _words_ in the IP header. The savviest hacker has already calculated (or knew) that because the field is 4-bit, it can only hold a maximum value of 15. Thus the maximum length of an IP header is 60 octets (15 times 32 divided by eight).

The _type of service_ field, `tos`, originates from the original IP specification[^ipv4-spec]. It has been divided into smaller fields in later specifications, but for simplicity's sake, we will treat the field as defined in the original specification. The field communicates the quality of service intended for the IP datagram.  

The _total length_, `len`, field communicates the length of the whole IP datagram. As it is a 16-bit field, its maximum value is 64 kilobytes (2 to the power of 16, minus 1, equalling 65535 bits).

The `id` field is used to index the datagram and is ultimately used for reassembly of fragmented IP datagrams. The field's value is simply a counter that is incremented by the sending party. In turn, the receiving end then knows how to order the incoming datagrams.

The `flags` field defines various control flags of the datagram. In specific, the sender can specify whether the datagram is allowed to be fragmented, whether it is the last fragment or that there's more fragments incoming.

The _fragment offset_ field, `frag_offset`, indicates the position of the fragment in a datagram. Naturally, the first datagram has this index set to 0.

The `ttl` or `time-to-live` is a common attribute that is used to count down the datagram's lifetime. It is usually set to 64 by the original sender, and every receiver decrements this counter by one. When it hits zero, the datagram is to be discarded and possibly an ICMP message is replied back to indicate an error.

The `proto` field provides the datagram an inherent ability to carry other protocols in its payload. The field usually contains values such as 16 (UDP) or 6 (TCP), and is simply used to communicate the type of the actual data to the receiver. 

The _header checksum_ field, `csum`, is used to verify the integrity of the IP header. The algorithm for it is relatively simple, and will be explained further down in this tutorial.

Finally, the `saddr` and `daddr` fields indicate the source and destination addresses of the datagram, respectively. Even though the fields are 32-bit in length and can thus depict approximately 4.5 billion addresses, the address range is still too small for our communication-centric society. The IPv6 protocol extends this length to 128-bits and as a result, future-proofs the address range of the Internet Protocol, perhaps permanently.

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

1. Adding the fields together yields the two's complement sum `1 1b 3e`.
2. Then, to convert it to one's complement, the carry-over bits are added to the first 16-bits: `1b 3e` + `1` = `1b 3f`.
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

The `csum` field is the same checksum field as in the IPv4 header, and the same algorithm can be used to calculate it. In ICMPv4 however, the payload is also included into the checksum.

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

The `data` field is optional, but often contains information like the timestamp of the echo. This can then be used to estimate the round-trip time between hosts..

## Checksum

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

# Sources
[^tcp-roadmap]:<https://tools.ietf.org/html/rfc7414>
[^ipv4-spec]:<http://tools.ietf.org/html/rfc791>
[^icmpv4-spec]:<https://www.ietf.org/rfc/rfc792.txt>
[^internet-checksum-spec]:<https://tools.ietf.org/html/rfc1071>
[^iana-icmp-types]:<http://www.iana.org/assignments/icmp-parameters/icmp-parameters.xhtml>

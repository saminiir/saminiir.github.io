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

# IPv4

## IP Header Format

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

## IP Fragmentation 

Test 

## IP Checksum

Test

# ICMPv4

# Conclusion

# Sources
[^tcp-roadmap]:<https://tools.ietf.org/html/rfc7414>
[^ipv4-spec]:<http://tools.ietf.org/html/rfc791>

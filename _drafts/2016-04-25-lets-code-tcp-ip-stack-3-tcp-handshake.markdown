---
layout: post
title:  "Let's code a TCP/IP stack, 3: TCP Handshake"
date:   2016-04-25 09:00:00
categories: [tcp/ip, tutorial, c programming, ip, icmp, networking, linux]
permalink: lets-code-tcp-ip-stack-3-tcp-handshake/
description: "This time in our tutorial userspace TCP/IP stack we will implement a minimum viable IP layer and test it with ICMP echo requests. We will take a look at the headers of IPv4 and ICMPv4 and describe how to check them for integrity. Some features, such as IP fragmentation, are left as an exercise for the reader."
---

This time in our userspace TCP/IP stack we will implement a minimum viable IP layer and test it with ICMP echo requests (also known as _pings_). 


packet corruption, packet reordering, packet duplication and packet erasures (drops). To amend this, TCP was designed to provide reliability to the transport layer.

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# Reliability mechanisms

The actual problem of sending data reliably may seem superficial, but its actual implementation is involved. Mainly, several questions arise regarding error-repair in a datagram-style network:

* How long should the sender wait for an acknowledgement from the receiver?
* What if the receiver cannot process data as fast as it is sent?
* What if the network in between (a router, for example) cannot process data as fast as it is sent?

In all of the scenarios, the underlying dangers of packet-switched networks apply - an acknowledgement from the receiver can be corrupted or even lost in transmit, which leaves the sender in a tricky situation.

To combat these problems, several mechanisms can be used. Perhaps the most common is the _sliding window_ technique, where both parties keep an account of the transmitted data. The window data is considered to be sequential (like a slice of an array) and that window "slides" forward as data is processed (and acknowledged) by both sides:

{% highlight bash %}

       Left window edge             Right window edge
             |                             |
             |                             |
---------------------------------------------------------
...|    3    |    4    |    5    |    6    |    7    |...
---------------------------------------------------------
        ^     ^                            ^    ^
        |      \                          /     |
        |       \                        /      |
   Sent and           Window size: 3         Cannot be
   ACKed                                     sent yet
   
{% endhighlight %}

The convenient property of using this kind of sliding window is that it also alleviates the problem of _flow control_. Flow control is required, when the receiver cannot process data as fast it is sent. In this scenario, the size of the sliding window would be negotiated to be lower, resulting in throttled output from the sender. 

# TCP Header Format

The TCP header is 20 octets in size[^tcp-spec]:

{% highlight bash %}
        0                            15                              31
       -----------------------------------------------------------------
       |          source port          |       destination port        |
       -----------------------------------------------------------------
       |                        sequence number                        |
       -----------------------------------------------------------------
       |                     acknowledgment number                     |
       -----------------------------------------------------------------
       |  HL   | rsvd  |C|E|U|A|P|R|S|F|        window size            |
       -----------------------------------------------------------------
       |         TCP checksum          |       urgent pointer          |
       -----------------------------------------------------------------
{% endhighlight %}

# Conclusion

{% include twitter.html %}

# Sources
[^tcp-roadmap]:<https://tools.ietf.org/html/rfc7414>
[^tcp-spec]:<https://www.ietf.org/rfc/rfc793.txt> 

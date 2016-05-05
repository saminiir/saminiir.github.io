---
layout: post
title:  "Let's code a TCP/IP stack, 3: TCP Basics & Handshake"
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

The problem of sending data reliably may seem superficial, but its actual implementation is involved. Mainly, several questions arise regarding error-repair in a datagram-style network:

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

The convenient property of using this kind of a sliding window is that it also alleviates the problem of _flow control_. Flow control is required, when the receiver cannot process data as fast it is sent. In this scenario, the size of the sliding window would be negotiated to be lower, resulting in throttled output from the sender. 

_Congestion control_, on the other hand, helps the networking stacks in between the sender and receiver to not get congested. There are two general methods for this: in the explicit version, the protocol has a field for specifically informing the sender about the congestion status. In the implicit version, the sender tries to guess when the network is congested and should throttle its output. Overall, congestion control is a complex, recurrent networking problem with accompanying research still being done to this day.[^stevens-tcpip]

# TCP Basics

TCP is a connection-oriented protocol.
TCP is a streaming protocol. 
TCP guarantees the application that data is received in-order.
TCP has a three-way handshake.

# TCP Header Format

As always, we'll define the message header and describe its fields. The TCP header is seemingly simple, but contains a lot of information about the communication state.

The TCP header is 20 octets in size[^tcpdump-man]:

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

The _Source Port_ and _Destination Port_ fields are used to establish multiple connections from and to hosts. Namely, the Berkeley sockets are the prevalent interface for applications to bind to the TCP networking stack. Through ports, the networking stack knows where to direct the traffic to. As the fields are 16 bits in size, the port values range from 0 to 65535.

Since every bytes in the stream is numbered, the _Sequence Number_ represents the  

The _Acknowledgment number_ contains the window's index of the next byte the sender expects to receive.

The _Header Length_ (HL) field presents the length of the header in 32-bit words.

Next, several flags are presented. The first 4 bits (_rsvd_) are not used.

Then, _Congestion Window Reduced_ (C) is used for informing that the sender reduced its sending rate.

_ECN Echo_ (E) informs that the sender received a congestion notification.

_Urgent Pointer_ (U) indicates that the segment contains prioritized data.

_ACK_ (A) field is used to communicate the state of the TCP handshake. It stays on for the remainder of the connection.

_PSH_ (P) is used to indicate that the receiver should "push" the data to the application as soon as possible.

_RST_ (R) resets the TCP connection.

_SYN_ (S) is used to synchronize sequence numbers in the initial handshake.

_FIN_ (F) indicates that the sender has finished sending data.

The _Window Size_ field is used to advertise the window size. In other words, this is the number of bytes the receiver is willing to accept. Since it is a 16-bit field, the maximum window size is 65,535 bytes.

The _TCP Checksum_ field is used to verify the integrity of the TCP segment. The algorithm is the same as for the Internet Protocol, but the input segment also contains the TCP data and also a pseudo-header from the IP datagram. 

The _Urgent Pointer_ is used when the U-flag is set. The pointer indicates the position of the urgent data in the stream.

After the header, several options can be provided. An example of these options is the _Maximum Segment Size_ (MSS), where the sender informs the other side of the maximum size of the segments.

After the possible options, the actual data follows. The data, however, is not required. For example, the handshake is accomplished with only TCP headers.

# Testing the TCP Handshake

# Conclusion

{% include twitter.html %}

# Sources
[^tcp-roadmap]:<https://tools.ietf.org/html/rfc7414>
[^tcp-spec]:<https://www.ietf.org/rfc/rfc793.txt> 
[^stevens-tcpip]:<https://en.wikipedia.org/wiki/TCP/IP_Illustrated#Volume_1:_The_Protocols>
[^tcpdump-man]:<http://www.tcpdump.org/tcpdump_man.html>

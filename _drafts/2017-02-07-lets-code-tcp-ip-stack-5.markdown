---
layout: post
title:  "Let's code a TCP/IP stack, 5: TCP Retransmission"
date:   2017-02-07 12:00:00
categories: [tcp/ip, tutorial, c programming, ip, networking, linux]
permalink: lets-code-tcp-ip-stack-5-tcp-retransmissionn/
description: ""
---

At this point we have a TCP/IP stack that is able to communicate to other hosts in the Internet. The implementation so far has been fairly straight-forward, but missing a major feature: Reliability.

Namely, our TCP does not guarantee the integrity of the data stream it presents to applications. Even establishing the connection can fail if the handshake packets are lost in transit.

Introducing reliability and control is our next main focus in creating a TCP/IP stack from scratch.

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# Automatic Repeat Request

The basis for many reliable protocols is the concept of Automatic Repeat reQuest (ARQ)[^arq]. 

In ARQ, a receiver sends acknowledgments for data it has received, and the sender retransmits data it never received acknowledgements for.

As we have discussed, TCP keeps the sequence numbers of transmitted data in memory and responds with acknowledgments. The transmitted data is put into a retransmission queue and timers associated with the data are started. If no acknowledgment for the sequence of data is received before the timer runs out, a retransmission occurs.

As can be seen, TCP builds its reliability on the principles of ARQ. The detailed implementations of ARQ are, however, very involved. Simple questions like "how long should the sender wait for an acknowledgement?" are tricky to answer, especially when maximum performance is desired. TCP extensions like Selective Acknowledgments (SACK)[^sack] alleviate efficiency problems by acknowledging out-of-order data and avoiding unnecessary round-trips.

# TCP Retransmission

Retransmissions in TCP are described in the original specification[^tcp-spec] as:

_When the TCP transmits a segment containing data, it puts a copy on a retransmission queue and starts a timer; when the acknowledgment for that data is received, the segment is deleted from the queue.  If the acknowledgment is not received before the timer runs out, the segment is retransmitted._

However, the original formula for calculating the retransmission timeout was deemed inadequate for operating in different network environments. The current "standard method"[^stevens-tcpip] was described by Jacobson[^cong-avoid] and the latest codified specification can be found from RFC6298[^computing-timer].

The basic algorithm is relatively straightforward. For a given TCP sender, state variables are defined for calculating the timeout: 

* `rto` holds the actual measured timer length for the retransmission timeout
* `srtt` is _smoothed round-trip time_ to average the RTT over time
* `rttvar` is _round-trip time variation_ and is used to estimate the RTT variation

In short, `srtt` acts as a low-pass filter for the average RTT. Due to possible large variation in the RTT, `rttvar` is used to detect those changes and prevent them from skewing the averaging function. Additionally, a clock granularity of `G` seconds is assumed.

As described in RFC6298[^computing-timer], the steps for computation are as follows:

1. Before the first RTT measurement:
{% highlight bash %}
rto = 1000ms
{% endhighlight %}

2. On the first RTT measurement _R_:
{% highlight bash %}
srtt = R
rttvar = R/2
rto = srtt + max(G, 4*rttvar)
{% endhighlight %}
3. On subsequent measurements:
{% highlight bash %}
alpha = 0.125
beta = 0.25
rttvar = (1 - beta) * rttvar + beta * abs(srtt - r)
srtt = (1 - alpha) * srtt + alpha * r
rto = srtt + max(g, 4*rttvar)
{% endhighlight %}
4. After computing `rto`, if it is less than 1 second, round it up to 1 second. A maximum amount can be provided but it has to be at least 60 seconds

The clock granularity of TCP implementors has traditionally been estimated to be fairly high, ranging from 500ms to 1 second. Modern systems like Linux, however, use a clock granularity of 1 millisecond[^stevens-tcpip].

One thing to note is the RTO is suggested to always be at least 1 second. This is to guard against _spurious retransmissions_, i.e. when a segment is retransmitted too soon causing congestion. In practice, many implementations go for sub-second rounding: Linux uses 200ms.

# Karn's Algorithm

One mandatory algorithm that prevents the RTT measurement from giving false results is _Karn's Algorithm_[^karns-algorithm]. It simply states that the RTT samples should not be measured for retransmitted packets. 

In other words, the TCP sender keeps track whether the segment it sent was a retransmission and skips the RTT routine for those acknowledgments. This makes sense, since otherwise the RTO computation could easily be skewed and cause congestion.

The timestamp TCP option, however, can be used to measure RTT for every ACK segment. We will deal with the TCP Timestamp option in a separate blog post.

# Managing the RTO timer

Managing the retransmission timer is relatively straightforward. RFC6298 recommends the following algorithm:

1. When sending a data segment and the RTO timer is not running, activate it with the timeout value of `rto`
2. When outstanding data segments have been acknowledged, turn off the RTO timer
3. When an ACK is received for new data, restart the RTO timer with the value of `rto`

And when the RTO timer expires:

1. Retransmit the earliest unacknowledged segment 
2. Back off the RTO timer with a factor of 2, i.e. (`rto = rto * 2`)
3. Start the RTO timer

Additionally, when the backing off of the RTO value occurs and a subsequent measurement is successfully made, the RTO value can shrink drastically. The TCP implementation may clear `srtt` and `rttvar` when backing off and waiting for an acknowledgment.

# Signaling retransmission

A TCP is usually not only relying on the TCP sender's timers to fix lost packets. The receiving side can also inform the sender that segments need to be retransmitted.

_Duplicate acknowledgment_ is an algorithm where out-of-sequence segments are acknowledged, but by the sequence number of the latest in-order segment. After three duplicate acknowledgments, the TCP sender should realize that it needs to retransmit the segment that was advertised by the duplicate acknowledgments.

Furthermore, _Selective Acknowledgment_ (SACK) is a more sophisticated version of the duplicate acknowledgment. It is a TCP option where the receiver is able to encode the received sequences into its acknowledgements. Then the sender immediately notices any lost segments and resends them. We will discuss the SACK TCP option in a later blog post.

# Testing the retransmission implementation

# Conclusion

We have now essentially implemented a rudimentary TCP with simple data management and provided an interface applications can use.

However, TCP data communication is not a simple problem. The packets can get corrupted, reordered or lost in transit. Furthermore, the data transmit can congest arbitrary elements in the network.

For this, the TCP data communication needs to include more sophisticated logic. In the next post, we will look into TCP Window Management and TCP Retransmission Timeout mechanisms, to better cope with more challenging settings.

The source code for the project is hosted at [GitHub](https://github.com/saminiir/level-ip).

{% include twitter.html %}

# Sources
[^tcp-roadmap]:<https://tools.ietf.org/html/rfc7414>
[^tcp-spec]:<https://www.ietf.org/rfc/rfc793.txt> 
[^stevens-tcpip]:<https://en.wikipedia.org/wiki/TCP/IP_Illustrated#Volume_1:_The_Protocols>
[^tcpdump-man]:<http://www.tcpdump.org/tcpdump_man.html>
[^first-tcp-spec]:<https://tools.ietf.org/html/rfc675>
[^tcp-illustrated-implementation]:<http://www.kohala.com/start/tcpipiv2.html>
[^man-tcp]:<https://linux.die.net/man/7/tcp>
[^socket-man]:<http://man7.org/linux/man-pages/man7/socket.7.html>
[^arq]:<https://en.wikipedia.org/wiki/Automatic_repeat_request>
[^sack]:<https://tools.ietf.org/html/rfc2018>
[^cong-avoid]:<http://ee.lbl.gov/papers/congavoid.pdf>
[^computing-timer]:<https://tools.ietf.org/html/rfc6298>
[^karns-algorithm]:<https://en.wikipedia.org/wiki/Karn%27s_Algorithm>

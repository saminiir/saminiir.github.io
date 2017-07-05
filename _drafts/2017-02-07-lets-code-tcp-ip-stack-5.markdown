---
layout: post
title:  "Let's code a TCP/IP stack, 5: TCP Retransmission"
date:   2017-07-07 10:00:00
categories: [tcp/ip, tutorial, c programming, ip, networking, linux]
permalink: lets-code-tcp-ip-stack-5-tcp-retransmissionn/
description: "At this point we have a TCP/IP stack that is able to communicate to other hosts in the Internet. The implementation so far has been fairly straight-forward, but missing a major feature: Reliability. Namely, our TCP does not guarantee the integrity of the data stream it presents to applications. Even establishing the connection can fail if the handshake packets are lost in transit. Introducing reliability and control is our next main focus in creating a TCP/IP stack from scratch."
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

As can be seen, TCP builds its reliability on the principles of ARQ. However, detailed implementations of ARQ are involved. Simple questions like "how long should the sender wait for an acknowledgement?" are tricky to answer, especially when maximum performance is desired. TCP extensions like Selective Acknowledgments (SACK)[^sack] alleviate efficiency problems by acknowledging out-of-order data and avoiding unnecessary round-trips.

# TCP Retransmission

Retransmissions in TCP are described in the original specification[^tcp-spec] as:

_When the TCP transmits a segment containing data, it puts a copy on a retransmission queue and starts a timer; when the acknowledgment for that data is received, the segment is deleted from the queue.  If the acknowledgment is not received before the timer runs out, the segment is retransmitted._

However, the original formula for calculating the retransmission timeout was deemed inadequate for different network environments. The current "standard method"[^stevens-tcpip] was described by Jacobson[^cong-avoid] and the latest codified specification can be found from RFC6298[^computing-timer].

The basic algorithm is relatively straightforward. For a given TCP sender, state variables are defined for calculating the timeout: 

* `srtt` is _smoothed round-trip time_ for averaging the round-trip time (RTT) of a segment
* `rttvar` holds the _round-trip time variation_
* `rto` eventually holds the _retransmission timeout_, e.g. in milliseconds

In short, `srtt` acts as a low-pass filter for consecutive RTTs. Due to possible large variation in the RTT, `rttvar` is used to detect those changes and prevent them from skewing the averaging function. Additionally, a clock granularity of `G` seconds is assumed.

As described in RFC6298[^computing-timer], the steps for computation are as follows:

1. Before the first RTT measurement:
```ruby
rto = 1000ms
```
2. On the first RTT measurement _R_:
```ruby
srtt = R
rttvar = R/2
rto = srtt + max(G, 4*rttvar)
```
3. On subsequent measurements:
```ruby
alpha = 0.125
beta = 0.25
rttvar = (1 - beta) * rttvar + beta * abs(srtt - r)
srtt = (1 - alpha) * srtt + alpha * r
rto = srtt + max(g, 4*rttvar)
```
4. After computing `rto`, if it is less than 1 second, round it up to 1 second. A maximum amount can be provided but it has to be at least 60 seconds

The clock granularity of TCP implementors has traditionally been estimated to be fairly high, ranging from 500ms to 1 second. Modern systems like Linux, however, use a clock granularity of 1 millisecond[^stevens-tcpip].

One thing to note is that the RTO is suggested to always be at least 1 second. This is to guard against _spurious retransmissions_, i.e. when a segment is retransmitted too soon, causing congestion in the network. In practice, many implementations go for sub-second rounding: Linux uses 200 milliseconds.

# Karn's Algorithm

One mandatory algorithm that prevents the RTT measurement from giving false results is _Karn's Algorithm_[^karns-algorithm]. It simply states that the RTT samples should not be taken for retransmitted packets. 

In other words, the TCP sender keeps track whether the segment it sent was a retransmission and skips the RTT routine for those acknowledgments. This makes sense, since otherwise the sender could not distinguish acknowledgements between the original and retransmitted segment.

When utilizing the timestamp TCP option, however, the RTT can be measured for every ACK segment. We will deal with the TCP Timestamp option in a separate blog post.

# Managing the RTO timer

Managing the retransmission timer is relatively straightforward. RFC6298 recommends the following algorithm:

1. When sending a data segment and the RTO timer is not running, activate it with the timeout value of `rto`
2. When all outstanding data segments have been acknowledged, turn off the RTO timer
3. When an ACK is received for new data, restart the RTO timer with the value of `rto`

And when the RTO timer expires:

1. Retransmit the earliest unacknowledged segment 
2. Back off the RTO timer with a factor of 2, i.e. (`rto = rto * 2`)
3. Start the RTO timer

Additionally, when the backing off of the RTO value occurs and a subsequent measurement is successfully made, the RTO value can shrink drastically. The TCP implementation may clear `srtt` and `rttvar` when backing off and waiting for an acknowledgment[^computing-timer].

# Requesting retransmission

A TCP is usually not only relying on the TCP sender's timers to fix lost packets. The receiving side can also inform the sender that segments need to be retransmitted.

_Duplicate acknowledgment_ is an algorithm where out-of-sequence segments are acknowledged, but by the sequence number of the latest in-order segment. After three duplicate acknowledgments, the TCP sender should realize that it needs to retransmit the segment that was advertised by the duplicate acknowledgments.

Furthermore, _Selective Acknowledgment_ (SACK) is a more sophisticated version of the duplicate acknowledgment. It is a TCP option where the receiver is able to encode the received sequences into its acknowledgements. Then the sender immediately notices any lost segments and resends them. We will discuss the SACK TCP option in a later blog post.

# Trying it out

Now that we've gone through the concepts and general algorithms, let's see how TCP retransmissions look like on the wire.

Let's change the firewall rules to drop packets after the connection is established and try to fetch the front page of HN:

{% highlight bash %}
$ iptables -I FORWARD --in-interface tap0 \
	-m connbytes --connbytes 3 --connbytes-dir both \
	--connbytes-mode packets -j DROP
$ ./tools/level-ip curl news.ycombinator.com
curl: (56) Recv failure: Connection timed out
{% endhighlight %}

Observing the connection traffic we see that the `HTTP GET` is retransmitted with roughly a doubling interval:

{% highlight bash %}
[saminiir@localhost ~]$ sudo tcpdump -i tap0 host 10.0.0.4 -n -ttttt
00:00:00.000000 IP 10.0.0.4.41733 > 104.20.44.44.80: Flags [S], seq 3975419138, win 44477, options [mss 1460], length 0
00:00:00.004318 IP 104.20.44.44.80 > 10.0.0.4.41733: Flags [S.], seq 4164704437, ack 3975419139, win 29200, options [mss 1460], length 0
00:00:00.004534 IP 10.0.0.4.41733 > 104.20.44.44.80: Flags [.], ack 1, win 44477, length 0
00:00:00.011039 IP 10.0.0.4.41733 > 104.20.44.44.80: Flags [P.], seq 1:85, ack 1, win 44477, length 84: HTTP: GET / HTTP/1.1
00:00:01.094237 IP 104.20.44.44.80 > 10.0.0.4.41733: Flags [S.], seq 4164704437, ack 3975419139, win 29200, options [mss 1460], length 0
00:00:01.094479 IP 10.0.0.4.41733 > 104.20.44.44.80: Flags [.], ack 1, win 44477, length 0
00:00:01.210787 IP 10.0.0.4.41733 > 104.20.44.44.80: Flags [P.], seq 1:85, ack 1, win 44477, length 84: HTTP: GET / HTTP/1.1
00:00:03.607225 IP 10.0.0.4.41733 > 104.20.44.44.80: Flags [P.], seq 1:85, ack 1, win 44477, length 84: HTTP: GET / HTTP/1.1
00:00:08.399056 IP 10.0.0.4.41733 > 104.20.44.44.80: Flags [P.], seq 1:85, ack 1, win 44477, length 84: HTTP: GET / HTTP/1.1
00:00:18.002415 IP 10.0.0.4.41733 > 104.20.44.44.80: Flags [P.], seq 1:85, ack 1, win 44477, length 84: HTTP: GET / HTTP/1.1
00:00:37.289491 IP 10.0.0.4.41733 > 104.20.44.44.80: Flags [P.], seq 1:85, ack 1, win 44477, length 84: HTTP: GET / HTTP/1.1
00:01:15.656151 IP 10.0.0.4.41733 > 104.20.44.44.80: Flags [P.], seq 1:85, ack 1, win 44477, length 84: HTTP: GET / HTTP/1.1
00:02:32.590664 IP 10.0.0.4.41733 > 104.20.44.44.80: Flags [P.], seq 1:85, ack 1, win 44477, length 84: HTTP: GET / HTTP/1.1
{% endhighlight %}

Verifying the retransmission back-off and the receiver going silent is easy, but what about the scenario when retransmissions are triggered only for some segments? For optimal performance, the RTO algorithm needs to 'bounce back' when the connection is detected as healthy.

Let's set the firewall rule to only block the 6th packet for 6000 bytes:

{% highlight bash %}
$ iptables -I FORWARD --in-interface tap0 \
	-m connbytes --connbytes 6 \
	--connbytes-dir original --connbytes-mode packets \
	-m quota --quota 6000 -j DROP
{% endhighlight %}

Now, if we try to send some data, our TCP has to recognize the communication blackout and when it ends. Let's send 6009 bytes:

{% highlight bash %}
$ ./tools/level-ip curl -X POST http://httpbin.org/post \
	-d "payload=$(perl -e "print 'lorem ipsum ' x500")"
{% endhighlight %}

Let's step through the connection phases, see when retransmissions are triggered and how the RTO value changes. Below is a modified `tcpdump` output with inline comments about the TCP socket's internal state:

{% highlight bash %}
00.000000 10.0.0.4.49951 > httpbin.org.80: [S], seq 1, options [mss 1460]
00.120709 httpbin.org.80 > 10.0.0.4.49951: [S.], seq 1, ack 1, options [mss 8961]
00.120951 10.0.0.4.49951 > httpbin.org.80: [.], ack 1

- Connection established, TCP RTO value of 10.0.0.4:49951 is 1000 milliseconds.

00.122686 10.0.0.4.49951 > httpbin.org.80: [P.], seq 1:174, ack 1: POST /post
00.242564 httpbin.org.80 > 10.0.0.4.49951: [.], ack 174
01.141287 10.0.0.4.49951 > httpbin.org.80: [.], seq 174:1634, ack 1: HTTP
01.141386 10.0.0.4.49951 > httpbin.org.80: [.], seq 1634:3094, ack 1: HTTP
01.141460 10.0.0.4.49951 > httpbin.org.80: [.], seq 3094:4554, ack 1: HTTP
01.263301 httpbin.org.80 > 10.0.0.4.49951: [.], ack 1634
01.265995 httpbin.org.80 > 10.0.0.4.49951: [.], ack 3094

- So far so good, the remote host has acked our HTTP POST 
  and the start of our payload. RTO value is 336ms.

01.526797 10.0.0.4.49951 > httpbin.org.80: [.], seq 3094:4554, ack 1: HTTP
02.259425 10.0.0.4.49951 > httpbin.org.80: [.], seq 3094:4554, ack 1: HTTP
03.735553 10.0.0.4.49951 > httpbin.org.80: [.], seq 3094:4554, ack 1: HTTP

- The communication blackout caused by our iptables rule has started.
  Our TCP has to retransmit the segment multiple times. 
  The RTO value of the socket keeps increasing:
    01.526797: 618ms
    02.259425: 1236ms
    03.735553: 2472ms

06.692867 10.0.0.4.49951 > httpbin.org.80: [.], seq 3094:4554, ack 1: HTTP
06.819115 httpbin.org.80 > 10.0.0.4.49951: [.], ack 4554

- Finally the remote host responds. Our RTO value has increased to 4944ms.
  Karn\'s Algorithm takes effect here: The new RTO value cannot be measured
  with the retransmitted segment, so we skip it.

06.819356 10.0.0.4.49951 > httpbin.org.80: [.], seq 4554:6014, ack 1: HTTP
06.819442 10.0.0.4.49951 > httpbin.org.80: [P.], seq 6014:6182, ack 1: HTTP
06.948678 httpbin.org.80 > 10.0.0.4.49951: [.], ack 6014
06.948917 httpbin.org.80 > 10.0.0.4.49951: [.], ack 6182

- Now we get ACKs on the first try and the network is relatively healthy again.
  Karn\'s Algorithm allows us to measure the new RTO:
    06.948678: 309ms
    06.948917: 309ms
  
06.948942 httpbin.org.80 > 10.0.0.4.49951: [P.], seq 1:26, ack 6182: HTTP 100 Continue
06.949014 httpbin.org.80 > 10.0.0.4.49951: [.], seq 26:1486, ack 6182: HTTP/1.1 200 OK
06.949145 10.0.0.4.49951 > httpbin.org.80: [.], ack 26
06.949816 httpbin.org.80 > 10.0.0.4.49951: [.], seq 1486:2946, ack 6182: HTTP
06.949894 httpbin.org.80 > 10.0.0.4.49951: [.], seq 2946:4406, ack 6182: HTTP
06.950029 10.0.0.4.49951 > httpbin.org.80: [.], ack 2946
06.950030 httpbin.org.80 > 10.0.0.4.49951: [.], seq 4406:5866, ack 6182: HTTP
06.950161 httpbin.org.80 > 10.0.0.4.49951: [P.], seq 5866:6829, ack 6182: HTTP
06.950287 10.0.0.4.49951 > httpbin.org.80: [.], ack 5866
06.950435 10.0.0.4.49951 > httpbin.org.80: [.], ack 6829
06.958155 10.0.0.4.49951 > httpbin.org.80: [F.], seq 6182, ack 6829
07.082998 httpbin.org.80 > 10.0.0.4.49951: [F.], seq 6829, ack 6183
07.083253 10.0.0.4.49951 > httpbin.org.80: [.], ack 6830

- The data communication and connection is finished.
  No significant changes to the RTO measurement occur.
{% endhighlight %}

# Conclusion

Retransmissions in TCP is an essential part of a robust implementation. A TCP has to survive and be performant in changing network environments, where for example the latencies can suddenly spike or the network path is blocked for a moment.

Next time, we will take a look at TCP Congestion Control for achieving maximum performance without degrading the network's health.

In the meantime, try the project out. See [Getting Started](https://github.com/saminiir/level-ip/blob/master/Documentation/getting-started.md) on how to use it with cURL and Firefox!

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

---
layout: post
title:  "Let's code a TCP/IP stack, 4: TCP Data Flow & Socket API"
date:   2016-10-08 09:00:00
categories: [tcp/ip, tutorial, c programming, ip, networking, linux]
permalink: lets-code-tcp-ip-stack-4-tcp-data-flow-socket-api/
description: ""
---

Previously, we introduced ourselves to the TCP header and how a connection is established between two parties.

In this post, we will look into the flow of TCP data communication and how it is managed.

Additionally, we will provide an interface from the networking stack that applications can use for network communication. This _Socket API_ is then utilized in our application that sends a HTTP GET request to the given network host and reads the response.

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}



# Transmission Control Block

It is beneficial to start the discussion on TCP data management by defining the variables that record the data flow state.

In short, the TCP has to keep track of the sequences it has sent and received acknowledgments for. To achieve this, a data structure called the _Transmission Control Block_ (TCB) is generated for every opened connection.

The variables for the outgoing (sending) data are:

{% highlight bash %}
    Send Sequence Variables
	
      SND.UNA - send unacknowledged
      SND.NXT - send next
      SND.WND - send window
      SND.UP  - send urgent pointer
      SND.WL1 - segment sequence number used for last window update
      SND.WL2 - segment acknowledgment number used for last window update
	  ISS     - initial send sequence number
{% endhighlight %}

In turn, the following data is recorded for the receiving side:

{% highlight bash %}
    Receive Sequence Variables
											  
      RCV.NXT - receive next
      RCV.WND - receive window
      RCV.UP  - receive urgent pointer
      IRS     - initial receive sequence number
{% endhighlight %}

Additionally, helper variables of the current segment being processed are defined:

{% highlight bash %}
    Current Segment Variables
	
      SEG.SEQ - segment sequence number
      SEG.ACK - segment acknowledgment number
      SEG.LEN - segment length
      SEG.WND - segment window
      SEG.UP  - segment urgent pointer
      SEG.PRC - segment precedence value
{% endhighlight %}

Together, these variables constitute most of the TCP control logic for a given connection.

# TCP Data Communication

Once a connection is established, explicit handling of the data flow is taken into use. Three variables from the TCB are the most important for basic tracking of the state: 

* `SND.NXT` - The sender will track the next sequence number to use in `SND.NXT`. 
* `RCV.NXT` - The receiver records the next sequence number to expect in `RCV.NXT`.
* `SND.UNA` - The sender will record the *oldest* unacknowledged sequence number in `SND.UNA`. 

Given a sufficient time period, when no data transmit occurs, all these three variables will be equal.

For example, when A decides to send a segment with data to B, the following happens:

1. TCP A sends a segment and advances `SND.NXT` in its own records (TCB).

2. TCB B receives the segment and acknowledges it by advancing `RCV.NXT` and sends an ACK.

3. TCB A receives the ACK and advances `SND.UNA`.

The amount by which the variables are advanced is the length of the data in the segment.

This is the basis for TCP control logic over the transmit of data. Let's look at this more closely with `tcpdump`, a Linux utility to capture traffic on the wire:

{% highlight bash %}
[saminiir@localhost level-ip]$ sudo tcpdump -i any port 8000 -nt
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [S], seq 1525252, win 29200, length 0
IP 10.0.0.5.8000 > 10.0.0.4.12000: Flags [S.], seq 825056904, ack 1525253, win 29200, options [mss 1460], length 0
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [.], ack 1, win 29200, length 0
{% endhighlight %}

The address 10.0.0.4 initiates a connection from port 12000 to host 10.0.0.5 listening on port 8000.

After the three-way handshake, connection is established. Initial sequence numbers are 1525252 for 10.0.0.4:12000, and 825056904 for 10.0.0.5:8000.

{% highlight bash %}
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [P.], seq 1:18, ack 1, win 29200, length 17
IP 10.0.0.5.8000 > 10.0.0.4.12000: Flags [.], ack 18, win 29200, length 0
{% endhighlight %}
       					
The host 10.0.0.4:12000 sends a segment with 17 bytes of data, which 10.0.0.5:8000 acknowledges with an ACK segment. Relative sequence numbers are shown by default with tcpdump, so that's why `ack 18` is displayed. Internally, the TCP of the receiving host has advanced `RCV.NXT` with the number 17.

{% highlight bash %}
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [.], ack 1, win 29200, length 0
IP 10.0.0.5.8000 > 10.0.0.4.12000: Flags [P.], seq 1:18, ack 18, win 29200, length 17
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [.], ack 18, win 29200, length 0
IP 10.0.0.5.8000 > 10.0.0.4.12000: Flags [P.], seq 18:156, ack 18, win 29200, length 138
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [.], ack 156, win 29200, length 0
IP 10.0.0.5.8000 > 10.0.0.4.12000: Flags [P.], seq 156:374, ack 18, win 29200, length 218
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [.], ack 374, win 29200, length 0
{% endhighlight %}

{% highlight bash %}

IP 10.0.0.5.8000 > 10.0.0.4.12000: Flags [F.], seq 374, ack 18, win 29200, length 0

// 10.0.0.5:8000 informs that it has no more data to send (FIN flag)

IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [.], ack 375, win 29200, length 0
{% endhighlight %}

# TCP Socket API

To be able to utilize the networking stack, some kind of an interface has to be provided for applications. The _BSD Socket API_ is the most famous one and it originates from the 4.2BSD UNIX release from 1983.[^tcp-illustrated-implementation]

The basic API in Linux is pretty straight-forward. You reserve a socket by calling `socket(2)`, passing the type of the socket and protocol as parameters. Common values are `AF_INET` for the type and `SOCK_STREAM` as domain. This will default to a TCP-over-IPv4 socket. 

After succesfully reserving a TCP socket from the networking stack, you will connect it to a remote endpoint. This is where `connect(2)` is used and calling it will launch the TCP handshake.

From that point on, we can just `write(2)` and `read(2)` data from our socket. It is up to the application to determine what to do with the data.

The networking stack will handle queueing, retransmission, error-checking and reassembly of the data in the TCP stream. For the application, the inner acting of TCP is opaque. The only thing the application can be sure is that the TCP has acknowledged the responsibility of sending and receiving the stream of data.

# Testing our Socket API

Now that our networking stack provides a socket interface, we can write applications against it.

The popular tool `curl` is used to transmit data with a given protocol. We can replicate the HTTP GET behavior of `curl` by writing a minimal implementation:

{% highlight bash %}

$ ./lvl-ip curl 10.0.0.5

{% endhighlight %}

In the end, sending a HTTP GET request exercises the underlying networking stack only minimally.

# Conclusion

We have now essentially implemented a rudimentary TCP with simple data management and provided an interface applications can use.

However, TCP data communication is not a simple problem. The packets can get corrupted, reordered or lost in transit. Furthermore, the data transmit can congest various elements in the network.

For this, the TCP data communication needs to include more sophisticated logic. In the next post we will look into TCP Window Management and TCP Retransmission Timeout mechanisms, to better cope with more challenging settings.

The source code for the project is hosted at [GitHub](https://github.com/saminiir/level-ip).

{% include twitter.html %}

# Sources
[^tcp-roadmap]:<https://tools.ietf.org/html/rfc7414>
[^tcp-spec]:<https://www.ietf.org/rfc/rfc793.txt> 
[^stevens-tcpip]:<https://en.wikipedia.org/wiki/TCP/IP_Illustrated#Volume_1:_The_Protocols>
[^tcpdump-man]:<http://www.tcpdump.org/tcpdump_man.html>
[^tcp-seq-num-attack]:<http://www.ietf.org/rfc/rfc1948.txt>
[^osi-model]:<https://en.wikipedia.org/wiki/OSI_model>
[^first-tcp-spec]:<https://tools.ietf.org/html/rfc675>

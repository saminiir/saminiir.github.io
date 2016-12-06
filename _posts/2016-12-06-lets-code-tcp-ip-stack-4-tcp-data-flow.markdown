---
layout: post
title:  "Let's code a TCP/IP stack, 4: TCP Data Flow & Socket API"
date:   2016-12-06 12:00:00
categories: [tcp/ip, tutorial, c programming, ip, networking, linux]
permalink: lets-code-tcp-ip-stack-4-tcp-data-flow-socket-api/
description: "Previously, we introduced ourselves to the TCP header and how a connection is established between two parties. In this post, we will look into TCP data communication and how it is managed. Additionally, we will provide an interface from the networking stack that applications can use for network communication. This Socket API is then utilized by our example application to send a simple HTTP request to a website."
---

Previously, we introduced ourselves to the TCP header and how a connection is established between two parties.

In this post, we will look into TCP data communication and how it is managed.

Additionally, we will provide an interface from the networking stack that applications can use for network communication. This _Socket API_ is then utilized by our example application to send a simple HTTP request to a website.

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}



# Transmission Control Block

It is beneficial to start the discussion on TCP data management by defining the variables that record the data flow state.

In short, the TCP has to keep track of the sequences of data it has sent and received acknowledgments for. To achieve this, a data structure called the _Transmission Control Block_ (TCB) is initialized for every opened connection.

The variables for the outgoing (sending) side are:

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

Additionally, helper variables of the current segment being processed are defined followingly:

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

Once a connection is established, explicit handling of the data flow starts. Three variables from the TCB are important for basic tracking of the state: 

* `SND.NXT` - The sender will track the next sequence number to use in `SND.NXT`. 
* `RCV.NXT` - The receiver records the next sequence number to expect in `RCV.NXT`.
* `SND.UNA` - The sender will record the *oldest* unacknowledged sequence number in `SND.UNA`. 

Given a sufficient time period when TCP is managing the data communication and no transmit occurs, all these three variables will be equal.

For example, when A decides to send a segment with data to B, the following happens:

1. TCP A sends a segment and advances `SND.NXT` in its own records (TCB).

2. TCB B receives the segment and acknowledges it by advancing `RCV.NXT` and sends an ACK.

3. TCB A receives the ACK and advances `SND.UNA`.

The amount by which the variables are advanced is the length of the data in the segment.

This is the basis for TCP control logic over the transmit of data. Let's see how this looks like with `tcpdump(1)`, a popular utility for capturing network traffic:

{% highlight bash %}
[saminiir@localhost level-ip]$ sudo tcpdump -i any port 8000 -nt
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [S], seq 1525252, win 29200, length 0
IP 10.0.0.5.8000 > 10.0.0.4.12000: Flags [S.], seq 825056904, ack 1525253, win 29200, options [mss 1460], length 0
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [.], ack 1, win 29200, length 0
{% endhighlight %}

The address 10.0.0.4 (host A) initiates a connection from port 12000 to host 10.0.0.5 (host B) listening on port 8000.

After the three-way handshake, the connection is established and their internal TCP socket state is set to `ESTABLISHED`. Initial sequence numbers are 1525252 for A, and 825056904 for B.

{% highlight bash %}
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [P.], seq 1:18, ack 1, win 29200, length 17
IP 10.0.0.5.8000 > 10.0.0.4.12000: Flags [.], ack 18, win 29200, length 0
{% endhighlight %}
       					
Host A sends a segment with 17 bytes of data, which host B acknowledges with an ACK segment. Relative sequence numbers are shown by default with `tcpdump` to ease readability. Thus, `ack 18` is actually 1525253 + 17. 

Internally, the TCP of the receiving host (B) has advanced `RCV.NXT` with the number 17.

{% highlight bash %}
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [.], ack 1, win 29200, length 0
IP 10.0.0.5.8000 > 10.0.0.4.12000: Flags [P.], seq 1:18, ack 18, win 29200, length 17
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [.], ack 18, win 29200, length 0
IP 10.0.0.5.8000 > 10.0.0.4.12000: Flags [P.], seq 18:156, ack 18, win 29200, length 138
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [.], ack 156, win 29200, length 0
IP 10.0.0.5.8000 > 10.0.0.4.12000: Flags [P.], seq 156:374, ack 18, win 29200, length 218
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [.], ack 374, win 29200, length 0
{% endhighlight %}

The interplay of sending data and acknowledging it continues. As can be seen, the segments with `length 0` only have the ACK flag set, but the acknowledgement sequence numbers are precisely increment based on the previously received segment's length. 

{% highlight bash %}
IP 10.0.0.5.8000 > 10.0.0.4.12000: Flags [F.], seq 374, ack 18, win 29200, length 0
IP 10.0.0.4.12000 > 10.0.0.5.8000: Flags [.], ack 375, win 29200, length 0
{% endhighlight %}

Host B informs host A that it has no more data to send by generating a FIN segment. In turn, host A acknowledges this.

To finish the connection, host A also has to signal that it has no more data to send.

# TCP Connection Termination

Closing a TCP connection is likewise an involved operation, and can be forcibly terminated (RST) or finished with a mutual agreement (FIN).

The basic scenario is as follows:

1. The _active closer_ sends a FIN segment.
2. The _passive closer_ acknowledges this by sending an ACK segment.
3. The _passive closer_ starts its own close operation (when it has no more data to send) and effectively becomes an _active closer_.
4. Once both sides have sent a FIN to each other and they have acknowledged them to both directions, the connection closes.

Evidently, the closing of a TCP connection requires four segments, in contrast to the three segments of the TCP connection establishment (three-way handshake).

Additionally, TCP is a bi-directional protocol, so it is possible to have the other end announce it has no more data to send, but stay online for incoming data. This is called _TCP Half-close_.

The unreliable nature of packet-switched networks introduce additional complexity to the connection termination - FIN segments can disappear or never intentionally be sent, which leaves the connection in an awkward state. For example, in Linux the kernel parameter `tcp_fin_timeout` controls how many seconds TCP waits for a final FIN packet, before forcibly closing the connection. This is a violation of the specification, but is needed for Denial of Service (DoS) prevention.[^man-tcp]

Aborting a connection involves a segment with the RST flag set. Resets can occur because of many reasons, but some usual ones are:

1. Connection request to a nonexistent port or interface
2. The other TCP has crashed and ends up in a out-of-sync connection state
3. Attempts to disturb existing connections[^tcp-rst-attack]

Thus, the happy path of TCP data transmission never involves a RST segment.

# Socket API

To be able to utilize the networking stack, some kind of an interface has to be provided for applications. The _BSD Socket API_ is the most famous one and it originates from the 4.2BSD UNIX release from 1983.[^tcp-illustrated-implementation] The Socket API in Linux is compatible to the BSD Socket API.[^socket-man] 

A socket is reserved from the networking stack by calling `socket(2)`, passing the type of the socket and protocol as parameters. Common values are `AF_INET` for the type and `SOCK_STREAM` as domain. This will default to a TCP-over-IPv4 socket. 

After succesfully reserving a TCP socket from the networking stack, it will be connected to a remote endpoint. This is where `connect(2)` is used and calling it will launch the TCP handshake.

From that point on, we can just `write(2)` and `read(2)` data from our socket.

The networking stack will handle queueing, retransmission, error-checking and reassembly of the data in the TCP stream. For the application, the inner acting of TCP is mostly opaque. The only thing the application can rely on is that the TCP has acknowledged the responsibility of sending and receiving the stream of data, and that it will inform the application of unexpected behavior through the socket API.

As an example, let's look at the system calls that a simple invocation of `curl(1)` does:

{% highlight bash %}
$ strace -esocket,connect,sendto,recvfrom,close curl -s example.com > /dev/null
...
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("93.184.216.34")}, 16) = -1 EINPROGRESS (Operation now in progress)
sendto(3, "GET / HTTP/1.1\r\nHost: example.co"..., 75, MSG_NOSIGNAL, NULL, 0) = 75
recvfrom(3, "HTTP/1.1 200 OK\r\nCache-Control: "..., 16384, 0, NULL, NULL) = 1448
close(3)                                = 0
+++ exited with 0 +++
{% endhighlight %}

We observe the Socket API calls with `strace(1)`, a tool for tracing system calls and signals. The steps are:

1. A socket is opened with `socket`, and the type is specified as IPv4/TCP.

2. `connect` launches the TCP handshake. Destination address and port are passed to the function.

3. After the connection is successfully established, `sendto(3)` is used to write data to the socket - in this case, the usual HTTP GET request. 

4. From that point on, `curl` eventually reads the incoming data with `recvfrom`.

The astute reader may have noticed that no `read` or `write` system calls were placed. This is because the actual socket API does not contain those functions, but normal I/O operations can also be used. From `man socket(7)`:[^socket-man]

_In addition, the standard I/O operations like write(2), writev(2), sendfile(2), read(2), and readv(2) can be used to read and write data._

In the end, the Socket API contains multiple functions for just writing and reading data. This is complicated by the I/O family of functions, which can also be used to manipulate the socket's file descriptor.

# Testing our Socket API

Now that our networking stack provides a socket interface, we can write applications against it.

The popular tool `curl` is used to transmit data with a given protocol. We can replicate the HTTP GET behavior of `curl` by writing a minimal implementation:

{% highlight bash %}

$ ./lvl-ip curl example.com 80

...
<!doctype html>
<html>
<head>
    <title>Example Domain</title>

    <meta charset="utf-8" />
    <meta http-equiv="Content-type" content="text/html; charset=utf-8" />
</head>

<body>
<div>
    <h1>Example Domain</h1>
    <p>This domain is established to be used for illustrative examples in documents. You may use this
    domain in examples without prior coordination or asking for permission.</p>
    <p><a href="http://www.iana.org/domains/example">More information...</a></p>
</div>
</body>
</html>

{% endhighlight %}

In the end, sending a HTTP GET request exercises the underlying networking stack only minimally.

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
[^tcp-seq-num-attack]:<http://www.ietf.org/rfc/rfc1948.txt>
[^osi-model]:<https://en.wikipedia.org/wiki/OSI_model>
[^first-tcp-spec]:<https://tools.ietf.org/html/rfc675>
[^tcp-illustrated-implementation]:<http://www.kohala.com/start/tcpipiv2.html>
[^man-tcp]:<https://linux.die.net/man/7/tcp>
[^tcp-rst-attack]:<https://en.wikipedia.org/wiki/TCP_reset_attack>
[^socket-man]:<http://man7.org/linux/man-pages/man7/socket.7.html>

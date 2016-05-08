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

The underlying mechanisms in TCP are much more involved than in other protocols like UDP and IP. Namely, TCP is a _connection-oriented_ protocol, which means that an unicast communication channel is established between exactly two sides as a first step. This connection is being actively taken care of by both sides: Establishing the connection (Handshaking), informing the other party of the data's state and possible problems.

The other important property of TCP is that it is a _streaming_ protocol. Unlike UDP, TCP does not guarantee applications steady "chunks" of data as they are sent and received. Rather, the TCP implementation has to buffer the data and when packets get lost, reordered or corrupted, TCP has to wait and organize the the data in the buffer. Only when the data is deemed intact, TCP can hand the data over to the application's socket. 

As TCP operates on data as a stream, the "chunks" from the stream have to be converted into packets that IP can carry. This is called _packetization_, where the TCP header contains the sequence number of the current index in the stream. This also has the convenient property that the stream can be broken into many variable-sized segments and TCP then knows how to _repacketize_ them.

Similarly to IP, TCP also checks the message for integrity. This is achieved through the same checksum algorithm as in IP, but with added details. Mainly, the checksum is end-to-end, meaning that both the header and the data are included to the checksumming. In addition, a pseudo-header built from the IP header is included. 

If a TCP implementation receives corrupted segments, it discards them and does not notify the sender. This error is corrected by a timer set by the sender, which can be used to retransmit a segment if it was never acknowledged by the receiver. 

TCP is also a _full-duplex_ system, meaning that traffic can flow simultaneously in both direction. This means that the communicating sides have to keep the sequencing of data to both directions in memory. In more depth, TCP preserves its traffic footprint by including an acknowledgement for the opposite traffic in the segments it sends.

In essence, the sequencing of the data stream is the main principle of TCP. The problem of keeping it synchronized, however, is not a simple one. 

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

# TCP Handshake

A TCP connection usually undergoes the following phases: connection setup (handshaking), data transfer and closing of the connection. The following diagrams depicts the usual handshaking routine of TCP:

{% highlight bash %}
          TCP A                                                TCP B
    	  
    1.  CLOSED                                               LISTEN
    	
    2.  SYN-SENT    --> <SEQ=100><CTL=SYN>               --> SYN-RECEIVED
    	  
    3.  ESTABLISHED <-- <SEQ=300><ACK=101><CTL=SYN,ACK>  <-- SYN-RECEIVED
    			
    4.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK>       --> ESTABLISHED
    			  
    5.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK><DATA> --> ESTABLISHED

{% endhighlight %}
1. Host A's socket is in a closed state, which means it is not accepting connections. On the contrary, host B's socket that is bound to a specific port is listening for new connections.

2. Host A intends to initiate a connection with host B. Thus, A crafts a TCP segment that has its SYN flag set and also the Sequence field populated with a value (100).

3. Host B responds with a TCP segment that has its SYN and ACK fields set, and acknowledges A's Sequence number by adding 1 to it (ACK=101). Likewise, B generates a sequence number (300).

4. The 3-way handshake is finished by an ACK from the originator (A) of the connection request. The Acknowledgement field reflects the sequence number the host next expects to receive from the other side.

5. Data starts to flow, mainly because both sides have acknowledged each other's segment numbers.

This is the common scenario of establishing a TCP connection. However, several questions arise:

1. How is the initial Sequence number chosen?
2. What if both sides request a connection for each other at the same time? 
3. What if segments are delayed for some time or indefinitely? 

The _Initial Sequence Number_ (ISN) is chosen independently by both communication parties at first contact. As it is a crucial part of identifying the connection, it has to be chosen so that it is most likely unique and not easily guessable. Indeed, the _TCP Sequence Number Attack_[^tcp-seq-num-attack] is the situation when an attacker can replicate a TCP connection and effectively feeding data to the target.

The original specification suggests that the ISN is chosen by a counter that increments each 4 microseconds. This, however, can be guessed by an attacker. In reality, modern networking stacks generate the ISN by more complicated methods.

The situation where both endpoints receive a connection request (SYN) from each other is called a _Simultaneous Open_. This is solved by an extra message exchange in the TCP handshake: Both sides send an ACK (without knowing that the other side has done it as well), and both sides SYN-ACK the requests. After this, data transfer commences. 

Lastly, the TCP implementation has to have a timer for knowing when to give up on establishing a connection. Attempts are made to re-establish the connection, usually with an exponential backoff, but once the maximum retries or the time threshold is met, the connection is deemed to be non-existant

# Testing the TCP Handshake

Now that we have the TCP handshake routine mocked and it effectively listens for every port, let's test it:

{% highlight bash %}
[saminiir@localhost ~]$ nmap -Pn 10.0.0.4 -p 1337

Starting Nmap 7.00 ( https://nmap.org ) at 2016-05-08 19:02 EEST
Nmap scan report for 10.0.0.4
Host is up (0.00041s latency).
PORT     STATE SERVICE
1337/tcp open  waste

Nmap done: 1 IP address (1 host up) scanned in 0.05 seconds
{% endhighlight %}

Because of the fact that nmap does a SYN-scan (it only waits for the SYN-ACK to decide if the target's port is open), it is easy to fool it to think we have an application listening on the port by just returning a SYN-ACK TCP segment.

# Conclusion

The minimum viable TCP handshake routine can be done relatively effortless by just picking a Sequence number, setting the SYN-ACK flags and calculating the checksum for the resulting TCP segment. 

Next time, we'll start looking at the Berkeley Socket API. This is the prevalent interface for applications to bind to the host operating system's networking stack. We'll see and try if we can mock the socket API for a small application and thus use our networking stack to actually communicate.

We also have to look into the more complex parts of TCP: window management.

{% include twitter.html %}

# Sources
[^tcp-roadmap]:<https://tools.ietf.org/html/rfc7414>
[^tcp-spec]:<https://www.ietf.org/rfc/rfc793.txt> 
[^stevens-tcpip]:<https://en.wikipedia.org/wiki/TCP/IP_Illustrated#Volume_1:_The_Protocols>
[^tcpdump-man]:<http://www.tcpdump.org/tcpdump_man.html>
[^tcp-seq-num-attack]:<http://www.ietf.org/rfc/rfc1948.txt>

---
layout: post
title:  "Let's code a TCP/IP stack, 4: TCP Data Flow"
date:   2016-06-01 09:00:00
categories: [tcp/ip, tutorial, c programming, ip, networking, linux]
permalink: lets-code-tcp-ip-stack-4-tcp-data-flow/
description: ""
---

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# Recap of connection establishment

# TCP State Transition

# Managing the Window

In the previous post, we glossed over the 

# Transmission Control Block

To manage the TCP connection, both communicating parties need to remember several details about the connection's state.

This data structure is called the _Transmission Control Block_ (TCB), which is generated for every new opened connection.

The TCB holds important info for both directions of the connection. The variables for the outgoing (sending) data are:

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

{% highlight bash %}
	Receive Sequence Variables
											  
	  RCV.NXT - receive next
	  RCV.WND - receive window
	  RCV.UP  - receive urgent pointer
	  IRS     - initial receive sequence number
{% endhighlight %}



# Conclusion

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

---
layout: post
title:  "How does OpenVPN work?"
date:   2016-01-30 08:00:00
categories: networking
permalink: how-does-openvpn-work
---

I realized I do not know exactly *how* OpenVPN does its magic in Linux.

I mean, sure, it encrypts and tunnels the packets to another endpoint, but can you describe the exact steps to achieve this? If not, read on. In fact, read on anyway for good measure.

This is the general idea I had pictured in my mind how OpenVPN works:

* Receive packets
* Encrypt them
* Tunnel them
* Send to destination

Several questions arise, though:

* How is tunneling achieved in OpenVPN's case?
* What is the mechanism for receiving and sending the packets?
* What is the general idea behind secure communications in the public network?

Let's delve deeper into these questions.

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# Tunneling 

What is tunneling? According to the magnificent TCP/IP Illustrated[^1]:

_Tunneling, generally speaking, is the idea of carrying lower-layer traffic in higher-layer (or equal-layer) packets. For example, IPv4 can be carried in an IPv4 or IPv6 packet; Ethernet can be carried in a UDP or IPv4 or IPv6 packet, and so on._

In other words, tunneling is achieved by injecting packets into the payload of other packets. Reminds me of a certain meme.

Various protocols for establishing tunnels exist, the prominent ones being _Generic Routing Encapsulation_ (GRE), Microsoft's _Point-to-Point Tunneling Protoco_ (PPTP) and the _Layer 2 Tunneling Protocol_ (L2TP).

The net result is that virtual links between computers in different networks can be set up. An example of this is a company's private network, which can be reached from the public network via tunneling.

Note, however, that tunneling itself does not guarantee secure communications. Encryption has to be applied to the payload, assuming you've succeeded to exchange cryptographic keys securely in an insecure environment.

# Encryption

When using tunneling, there's a high chance that the information you transmit is of delicate nature. Hence, cryptographic measures have to be executed to prevent anyone from snooping in the data.

Since OpenVPN is an userspace application, _Transport Layer Security_ (TLS) is a natural choice for the security protocol[^2]. TLS operates in the layers above the networking stack, so implementing it in an application's scope is easier than more involved lower-level protocols that require specific kernel modules. If you've ever browsed a https-site, then you have used TLS for authentication and encryption.

Arguably the most important part of the TLS handshaking protocol is the key exchange. Key exchange is mandatory :w
is9 
The handshake part of the TLS protocol is somewhat involved.
The problem of negotiating a secure communication channel in a hostile environment is handled by various key exchange protocols. The Diffie-Hellman key exchange is perhaps the 
How does one start secure communications in

_OpenVPN uses an industrial-strength security model designed to protect against both passive and active attacks. OpenVPN's security model is based on using SSL/TLS for session authentication and the IPSec ESP protocol for secure tunnel transport over UDP. OpenVPN supports the X509 PKI (public key infrastructure) for session authentication, the TLS protocol for key exchange, the OpenSSL cipher-independent EVP interface for encrypting tunnel data, and the HMAC-SHA1 algorithm for
authenticating tunnel data._[^3]

TODO: Why does an asymmetric encryption scheme like X.509 need a calculated premaster secret?

DH

# OpenVPN implementation

## TUN/TAP devices

The tunneling of data in OpenVPN is achieved through TUN/TAP devices[^4]. Simply put, TUN/TAP devices expose the operating system's network traffic as virtual interfaces. This traffic can then be operated upon by the userspace application that is bound to the TUN/TAP virtual interface.

A TUN device operates on IP packets (layer 3), and a TAP device operates on ethernet frames (layer 2). The distinction is important, since operating on different networking layers enable different use cases. For example, if one wants Ethernet bridging, OpenVPN has to utilize TAP devices. For simple routing of traffic, TUN devices are a cheaper choice. A handy summary of the differences between bridging and routing is presented
[here](https://community.openvpn.net/openvpn/wiki/309-what-is-the-difference-between-bridging-and-routing).

How do packets get forwarded to/from the TUN/TAP device?

## Overview of OpenVPN in action

TODO: Describe step-by-step the OpenVPN setup and communication phases

{% highlight bash %}


{% endhighlight %}

# Conclusion

# Sources

[^1]:<https://en.wikipedia.org/wiki/TCP/IP_Illustrated#Volume_1:_The_Protocols>
[^2]:<https://community.openvpn.net/openvpn/wiki/WhyChooseTLSAsOpenvpnsUnderlyingAuthenticationAndKeyNegotiationProtocol>
[^3]:<https://community.openvpn.net/openvpn/wiki/OverviewOfOpenvpn>
[^4]:<https://www.kernel.org/doc/Documentation/networking/tuntap.txt>
[^5]:<https://community.openvpn.net/openvpn/wiki/309-what-is-the-difference-between-bridging-and-routing>

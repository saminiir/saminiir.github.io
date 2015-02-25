---
layout: post
title:  "Debugging the PXE protocol"
date:   2015-02-22 21:40:37
categories: meta
permalink: debugging-the-pxe-protocol
---

In my earlier post, I defined the steps for setting up a PXE boot environment.

However, it was not working straight off. I could see from my router's logs that the machine booting from LAN sent a DHCPDISCOVER message and the router responded with a DHCPOFFER. The machine never continued with a DHCPREQUEST however.


---
layout: post
title:  "Automate all the camera photo uploads, part I"
date:   2015-12-07 20:00:00
categories: raspberry-pi
permalink: automate-camera-photo-uploads-part-1
description: "So this fall I've been getting into photography and with my super small new camera (Lumix GM1) I'm looking forward to taking horrendous photos of unexpecting people. As the nerd I am, however, I deem that no tasks should be done repeatedly and manually. This is also true for transferring photos from camera to safety. Ideally, you would only power on the camera and the photos would just sync to your computer and/or cloud service. As the year is 2015 you would expect a pretty much seamless (and wireless) user experience for scenarios like this, but alas, that is not the case."
---

So this fall I've been getting into photography and with my super small new camera (Lumix GM1) I'm looking forward to taking horrendous photos of unexpecting people.

As the nerd I am, however, I deem that no tasks should be done repeatedly and manually. This is also true for transferring photos from camera to safety. Ideally, you would only power on the camera and the photos would just sync to your computer and/or cloud service. As the year is 2015 you would expect a pretty much seamless (and wireless) user experience for scenarios like this, but alas, that is not the case. At least on this Lumix 2013 model the WiFi is dog-slow and stubborn, and photos can not be
mass-transferred. I mean, what the hell?  

I came up with a better plan. Why not just plug in the camera to my always-online Raspberry Pi USB hub and let some UNIX magic take care of the photo backup and upload? This seems so much more straight-forward than interacting with the camera's user interface. Plus, the data transfer is much faster through USB compared to the camera's WiFi.

So let's get started. Connect your camera to the USB port and pick out the vendor and product ID's:

{% highlight bash %}
$ lsusb
Bus 001 Device 037: ID 04xc:23yz Panasonic (Matsushita) Lumix Camera
                       ^^^^ ^^^^
{% endhighlight %}

Add a hook to your `/etc/udev/rules.d/` so that a script gets run every time the camera gets connected:

{% highlight bash %}
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="04xc", ATTR{idProduct}=="23yz", RUN+="/home/pi/bin/photo_backup.sh"
{% endhighlight %}

Pay attention to the manpages of udev where it is stated that RUN "can only be used for very short running tasks." Long running processes are encouraged to "immediately detach from the event process itself."

Thus we'll spawn of a background process inside our scripts to not block udev's execution. We'll also use `rsync` for intelligent backing up of the photos:

{% highlight bash %}
$ cat /home/pi/bin/photo_backup.sh
#!/bin/sh

{
  PHOTO_FOLDER="/media/9016-4EF8/DCIM/"
  DESTINATION_FOLDER="/media/passport/photos/"
  THRESHOLD=20
  # Wait for photo folder to come available
  while [ ! -d $PHOTO_FOLDER -a $THRESHOLD -gt 0 ]; do
    THRESHOLD=$(( $THRESHOLD - 1 ))
    echo "Folder $PHOTO_FOLDER not yet available, threshold: $THRESHOLD"
    sleep 1
  done

  if [ $THRESHOLD -eq 0 ]; then
    echo "Threshold was met, folder $PHOTO_FOLDER never came available"
    exit 1
  fi

  rsync -av $PHOTO_FOLDER $DESTINATION_FOLDER
} &

{% endhighlight %}

And there we go, everytime you plug in your camera to the USB hub, the photos are automatically backed up.

If you do decide to go by the wireless approach, just run a Samba service on your Linux machine and save the camera's connection for fast access. 

In the next post, we'll automate sending the photos to a cloud service for easy access.

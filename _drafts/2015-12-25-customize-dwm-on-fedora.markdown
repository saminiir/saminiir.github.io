---
layout: post
title:  "Customize DWM on Fedora"
date:   2015-12-25 20:00:00
categories: fedora
permalink: customize-dwm-on-fedora
---

So at work I'm using a Macbook, but OS X has increasingly interfered with my development routine.

The last drop that tipped the glass was no native support for Docker in OS X, obviously.

Hence I'm moving to using a Linux VM in VMWare. I chose Fedora, because that's what Linus uses. 

Just kidding, I wanted to learn a Red Hat based distro this time, since Debian and Arch are already familiar to me.

After launching Fedora 23, I wanted to set up a tiling window manager. If you haven't used one, try the magnificent XMonad. This time I opted for dwm, however, since it is suppose to "suck less" and there's also strange appeal in light-weight C projects. The configuration of dwm is done by editing a configuration header in the C source files and by compiling the project. I could've just cloned the git repo and build the project myself, but I wanted to utilize the existing RPM package for dwm as an exercise.

A surprise in Fedora for me was that it had deprecated yum, the well-known package manager. Instead, it utilizes DaNdiFied yum, `dnf`. Let's use it to get started:

{% highlight bash %}
$ dnf install rpmbuild
$ dnf download --source dwm

$ rpm -ivh dwm-6.0-13.fc23.src.rpm
{% endhighlight %}

You'll end up with a ~/rpmbuild directory, which is a central location for storing RPM package builds. Next you'll want to reuse the package information:

{% highlight bash %}
$ cd ~/rpmbuild
$ rpmbuild -ba SPECS/dwm.spec
{% endhighlight %}

Now you'll have the folder ~/rpmbuild/BUILD/dwm-6.0, which you can edit, generate a patch and make RPM apply it on the build phase:

{% highlight bash %}

pushd BUILD/dwm-6.0 && git diff -p --relative config.def.h > dwm-customization.patch && popd

mv BUILD/dwm-6.0/dwm-customization.patch SOURCES/
{% endhighlight %}

Edit the SPEC/dwm.spec file to apply the patch:

{% highlight bash %}

what 
{% endhighlight %}

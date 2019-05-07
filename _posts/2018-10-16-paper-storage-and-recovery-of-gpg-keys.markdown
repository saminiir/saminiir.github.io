---
layout: post
title:  "Paper storage and recovery of GPG keys"
date:   2018-10-16 10:00:00
categories: [gpg, cryptography]
permalink: paper-storage-and-recovery-of-gpg-keys/
description: "Paper storage and recovery of GPG keys"
---

To avoid disastrously losing your private keys, they should be redundantly backed up in a robust manner in a safe location. 

As one possible approach, we will store the GPG keyring as QR codes, print them on paper, and demonstrate recovering the keyring. Do note that this method excludes backing up the trust database.

For general GPG usage, consult my [previous post on handling GPG keys](/establish-cryptographic-identity-using-gnupg/).

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# Dependencies

Consult the Internet or your distro's package mirrors for the following packages: 

- [gpg2](https://gnupg.org/)
- [ImageMagick](https://github.com/ImageMagick/ImageMagick)
- [qrencode](https://github.com/fukuchi/libqrencode)
- [zbar](http://zbar.sourceforge.net/)

# Exporting the key

First, we'll peruse gpg's manual pages about exporting keys:

{% highlight bash %}
--export-secret-keys
--export-secret-subkeys
       Same as --export, but exports the secret keys instead.  The exported keys are written to STDOUT or to the file given with option --output.  This command is often used along with the option  --armor  to  allow  for  easy
       printing  of  the  key  for paper backup; however the external tool paperkey does a better job of creating backups on paper.  Note that exporting a secret key can be a security risk if the exported keys are sent over an
       insecure channel.

{% endhighlight %}

Important point being, never transmit private keys over an insecure medium.

Export your private and public keys into a keyring:

{% highlight bash %}
$ gpg --export-secret-keys --export-options export-minimal > private-keyring.gpg
{% endhighlight %}

The reason I am not using `paperkey` is that it requires corresponding public keys to be stored elsewhere. I want both private and public keys stored on the paper for easier retrieval in case of emergency. 

# Backing up the key

Let's base64 encode the keyring and split it into smaller bytes because of QR code storage limitations[^qr-code-storage]:

{% highlight bash %}
$ base64 private-keyring.gpg > private-keyring.base64
$ split -d -C 2500 private-keyring.base64 splitkey-
{% endhighlight %}

Then, we'll use `qrencode`[^qrencode] to create QR code pictures:

{% highlight bash %}
$ for i in splitkey-*; do qrencode -r $i -o "$i.PNG"; done
{% endhighlight %}

Finally, concatenate the pictures into a single photo for printing purposes:

{% highlight bash %}
$ montage -mode Concatenate -tile 2x splitkey-*.PNG keyring-qr-codes.PNG
{% endhighlight %}

Then, print this on paper.

# Recovering the key

Now, what to do when this paper backup is your only option left?

Scan the QR codes from the paper as JPEG, for example (EDIT: Actually, do PNG[^png-correction]).

Use `zbarimg` to decode the QR codes as data:

{% highlight bash %}
$ zbarimg --raw qr-codes.jpg > keyring.base64
{% endhighlight %}

Your mileage will probably vary on the scan's success rate. I had to manually crop the QR codes into separate JPEGs in order to construct the resulting base64 in correct order. `zbarimg` also adds newlines to the scanned data, so essentially manual reconstruction of the base64 data is needed.

Now, you can import it into a gpg instance:

{% highlight bash %}
$ base64 -d keyring.base64 > keyring.gpg
$ mkdir -p .gnupg/private-keys-v1.d
$ gpg --homedir .gnupg --import keyring.gpg
$ gpg -K
{% endhighlight bash %}

You should now be able to decrypt any private data stored.

# Conclusion

This approach is just one solution for backing up your GPG keys and recovering them in case of emergency. Also, if disaster strikes, it is good to have some routine/documentation on the recovery actions. This article serves as such document.

{% include twitter.html %}

# Sources
[^paperkey]:<http://www.jabberwocky.com/software/paperkey/>
[^qrencode]:<https://github.com/fukuchi/libqrencode>
[^qr-code-storage]:<https://en.wikipedia.org/wiki/QR_code#Storage>
[^png-correction]:<https://news.ycombinator.com/item?id=18228408>

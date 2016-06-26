---
layout: post
title:  "Establish your online identity using GnuPG"
date:   2016-01-24 19:00:00
categories: cryptography
permalink: establish-cryptographic-identity-using-gnupg/
comments: false
description: "Creating and managing PGP keys is not a straightforward matter. Many approaches exist and if you are a whistleblower, this tutorial probably does not meet your security standards. In fact, if there's a chance you'll be captured, tortured, or killed for the information you'll encrypt, stop reading this tutorial and pray for the best. You are doing it wrong."
---

I have a confession to make.

I do not have an online identity. I have Twitter, Facebook (yeah I know) and this homepage, but still, my online presence is undermined by the fact that I or others cannot verify my identity.

The remedy to this is public-key cryptography[^1].

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# About PGP

Using GPG[^2], the OpenPGP-compliant[^3] cryptographic software suite, you are able to prove your identity online and that of others. The principle behind this is called the web of trust[^4], where users of OpenPGP-compatible software sign each others public signatures (keys) after having verified their identity. This is in contrast to X.509[^5], where a Certificate Authority, CA, is the sole party establishing the authenticity of users.

So in a way, PGP adheres more to the open source philosophy of decentralized mode-of-operation. Plus, [the story of PGP](https://en.wikipedia.org/wiki/Pretty_Good_Privacy#Criminal_investigation) is pretty bad-ass.

In this tutorial, we'll refer to the OpenPGP specification as "PGP" and "GPG" as the free software implementation by the GNU Project.

# Using GPG

Creating and managing PGP keys is not a straightforward matter. Many approaches exist and if you are a whistleblower, this tutorial probably does not meet your security standards. In fact, if there's a chance you'll be captured, tortured, or killed for the information you'll encrypt, stop reading this tutorial and pray for the best. You are doing it wrong.

For others, I think this tutorial serves as a suitable middle ground for the pragmatic daily usage of PGP.

# Creating the master key

Let's get started creating your first PGP-based identity:

{% highlight bash %}
$ gpg2 --version
gpg (GnuPG) 2.1.9
libgcrypt 1.6.4

Home: ~/.gnupg
Supported algorithms:
Pubkey: RSA, ELG, DSA, ECDH, ECDSA, EDDSA
Cipher: IDEA, 3DES, CAST5, BLOWFISH, AES, AES192, AES256, TWOFISH,
        CAMELLIA128, CAMELLIA192, CAMELLIA256
        Hash: SHA1, RIPEMD160, SHA256, SHA384, SHA512, SHA224
        Compression: Uncompressed, ZIP, ZLIB, BZIP2

$ gpg2 --full-gen-key
{% endhighlight %}

Pay attention to the fact that we're using GPG v2.1 in this tutorial. The newest version prefers better hash algorithms.

Select the default `RSA and RSA` for making keys for both signing and encrypting:

{% highlight bash %}
Pleasese select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
{% endhighlight %}

Next you will be asked to input the size for the RSA keys. We'll use 4096 bits for the master key, since it is only meant for signing other keys. The subkeys we'll use are used for encryption and decryption, so they are [better off](https://www.gnupg.org/faq/gnupg-faq.html#no_default_of_rsa4096) with a 2048 bit key size[^6].

{% highlight bash %}
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
{% endhighlight %}

Set an expiration time for your master key, in case you lose your revokation certificate and the master key is compromised. The downside is that you have to keep extending the lifetime of the key. The expiration time can be altered afterwards, even if a key has expired.

{% highlight bash %}
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)
{% endhighlight %}

Input your real name and email. Do not, however, [input a comment](https://www.debian-administration.org/users/dkg/weblog/97) when prompted:

{% highlight bash %}
GnuPG needs to construct a user ID to identify your key.

Real name: Sami Niiranen
Email address: saminiir@gmail.com
Comment:
You selected this USER-ID:
    "Sami Niiranen <saminiir@gmail.com>"
{% endhighlight %}

Next up, you have to come up with a passphrase for your master key. The Intercept has [an insighftul article](https://theintercept.com/2015/03/26/passphrases-can-memorize-attackers-cant-guess/) on the problems of passphrases and how to create robust ones (with Diceware).

After some dice rolling and as you have entered the passphrase, the master key will be generated. This master key IS your electronic identity, so the point is to handle it with care. Especially the private master key we'll evacuate to a safe place later on in this tutorial.

## Key management

The case is often that one has multiple computers (desktop, laptop) and wants to use the same GPG identity on all devices. One popular scheme for this is creating subkeys, which are then distributed. There are, however, some differing opinions[^11][^12] on this approach and whether it is a sane model for the web of trust.

The alternative is to create master keypairs for every device and sign them with your ultra-secure master-master keypair.

In this tutorial, the approach of creating subkeys is chosen, mostly because it seems to be the more popular[^13] choice and has the advantage of accumulating trust for your single PGP identity.

## Creating a revocation certificate

GnuPG version 2.1 automatically generates a revocation certificate. It will be located under `$GNUPGHOME/openpgp-revocs.d/`. If you're using an older version, be sure to generate one.

Store this revocation certificate in a safe place. The revocation certificate is short enough to be handily stored on paper.

## Creating subkeys

Subkeys are your day-to-day keys for signing and encryption. Hence, we'll generate a subkey to the very computer you are now at.

{% highlight bash %}
$ gpg2 --list-keys $yourid
$ gpg2 --edit-key $yourid
$ gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
Your selection? 4

{% endhighlight %}

Choose the `RSA (sign only)` option for generating a signing key only. Use the default 2048-bit keysize if you have no reason to use a larger one. Choose an expiration date. Input a passphrase.

Repeat the procedure, but this time, create a RSA (encrypt only) key.

As a final step, trust and save the keys you've made:

{% highlight bash %}
gpg> trust
gpg> save
{% endhighlight %}

After these operations, your keyring should look similar to this:

{% highlight bash %}
$ gpg2 -k
/home/saminiir/.gnupg/pubring.kbx
---------------------------------
pub   rsa4096/XXXXXXXX 2016-01-24 [expires: 2016-07-22]
uid         [ultimate] Sami Niiranen <saminiir@gmail.com>
sub   rsa4096/XXXXXXXX 2016-01-24 [expires: 2016-07-22]
sub   rsa2048/XXXXXXXX 2016-01-24 [expires: 2016-07-22]
sub   rsa2048/XXXXXXXX 2016-01-24 [expires: 2016-07-22]
{% endhighlight %}

## Evacuating the master key to safety

Now you should have a master keypair and a signing/encryption sub-keypair for your personal computer. It is imperative, that the master keypair is now evacuated to safety.

There are endless possibilities how to store your master keypair, from paper printouts in a safe, to armed guards in your basement. Because I haven't yet obtained any interesting information to whistleblow, I chose the mundane approach of backing up the master keypair to a USB drive:

{% highlight bash %}
$ gpg2 --export-secret-keys --armor $keyid > privkey
$ gpg2 --export --armor $keyid > pubkey
$ gpg2 --export-secret-subkeys $keyid > subkeys
$ gpg2 --delete-secret-key $keyid
$ gpg2 --delete-key $keyid
$ gpg2 --import subkeys
$ shred subkeys
{% endhighlight %}

Alternatively, with gpg2, you can delete keys without needing to twiddle with exporting and importing:

{% highlight bash %}
# Get the keygrip of the key to be deleted
$ gpg2 -K --with-keygrip
$ rm .gnupg/private-keys-v1.d/$keygrip.key
# Make sure the older gpg format, secring.gpg, is empty and does not contain the key
$ ls -l .gnupg/secring.gpg 
{% endhighlight %}

You can verify that the master secret key should not be in the keyring by detecting `sec#` when listing the secret keys.

Now, be sure to evacuate your master keypair `privkey` and `pubkey` with your chosen method to a safe location. I'm storing it in an encrypted USB drive. The short gist of that is as follows:

{% highlight bash %}
$ lsblk
$ mount /dev/sdb /mnt/usb
$ dd if=/dev/zero of=/mnt/usb/disk.img bs=1M count=16
$ loopdev=$(losetup -f)
$ losetup $loopdev /mnt/usb/disk.img
$ cryptsetup -v luksFormat $loopdev
$ cryptsetup isLuks $loopdev && echo "Success"
$ cryptsetup open $loopdev usbkey
$ mkfs.ext3 /dev/mapper/usbkey
$ mount /dev/mapper/usbkey /media/encrypted
$ cp -r ~/.gnupg /media/encrypted/
$ gpg2 --homedir /media/encrypted/.gnupg -K
$ fuser -km /media/encrypted
$ umount /media/encrypted
$ cryptsetup remove usbkey
$ losetup -d $loopdev
$ umount /mnt/usb
{% endhighlight %}

You can [automatize the mounting of the encrypted filesystem](https://help.ubuntu.com/community/GPGKeyOnUSBDrive).

## Sending the public key to a key server

Next, you'll want your public key to be actually obtainable for other people. Keyservers are a good fit for this, since the gpg tool can be used to query them for other people's public keys.

For enhanced security, obtain the CA cert for the keyserver:

{% highlight bash %}
$ wget https://sks-keyservers.net/sks-keyservers.netCA.pem
{% endhighlight %}

Set the following as your keyserver configuration in `$GNUGPHOME/dirmngr.conf`.

{% highlight bash %}
keyserver hkps://hkps.pool.sks-keyservers.net
hkp-cacert /path/to/CA/sks-keyservers.netCA.pem
{% endhighlight %}

Send your public key to the keyserver pool. Afterwards you and anybody else can find your PGP identity.

{% highlight bash %}
$ gpgconf --reload
$ gpg2 --send-key $keyid
{% endhighlight %}

And you're done setting up your GPG configuration! The following section depicts use cases and how to solve them.

# Scenarios

## Importing the primary secret key

Since you need the primary secret key for many operations, like signing new keys, it needs to be imported from the safe storage medium back to your keychain:

{% highlight bash %}
$ mount /dev/sdb /mnt/usb
$ losetup --find --show /mnt/usb/disk.img
/dev/loop0
$ cryptsetup open /dev/loop0 usb
$ mount /dev/mapper/usb /media/encrypted
{% endhighlight %}

Replace the devices file descriptors with the correct ones.

Then, import the primary secret key:

{% highlight bash %}
$ gpg2 --homedir /media/encrypted/.gnupg --export-secret-keys --armor $KEYID > privkey
$ gpg2 --import privkey
{% endhighlight %}

After finishing with key manipulation, delete the private key from the keychain:

{% highlight bash %}
# Get the keygrip of the key to be deleted
$ gpg2 -K --with-keygrip
$ rm .gnupg/private-keys-v1.d/$keygrip.key
# Make sure the older gpg format, secring.gpg, is empty and does not contain the key
$ ls -l .gnupg/secring.gpg 
{% endhighlight %}

## New subkey

Create a new subkey. You'll need your master keypair that is stashed away to certify the new subkey.

## Signing keys

Again, you need the master secret key for this operation.

Obtain, sign and export somebody's key:

{% highlight bash %}
$ gpg2 --search-keys $email #Choose the public key to import
$ gpg2 --fingerprint $email #Check the fingerprint
$ gpg2 --sign-key $email
$ gpg2 --armor --export $keyid > signedkey
$ gpg2 --sign --encrypt --recipient $email signedkey
{% endhighlight %}

And mail the resulting .gpg file to the person.

## Extending the lifetime of a key

The point of setting an expiration date for all of your keys is to have them actually expire if you lose access to the master keys or the revokation key. Besides, checking your keys at least once per year can is a good practice that probably enhances your security.

Just set the expiration date through the gpg interface:

{% highlight bash %}

$ gpg2 --edit-key saminiir@gmail.com
gpg> expire
{% endhighlight %}

Again, you need your master keypair for this operation to succeed.

## Revoking the master key

To revoke your key, simply import the revocation certificate and update keyservers:

{% highlight bash %}
# Read the cert, you need to modify it to prevent accidental revokement 
$ vim .gnupg/openpgp-revocs.d/revcert.rev
$ gpg2 --import revcert.rev
$ gpg2 --send $keyid
{% endhighlight %}

Do not delete revoked keys. They are still useful to decrypt data previously encrypted with the old key.

## Revoking a subkey

To revoke a subkey, revoke it from gpg's interface:

{% highlight bash %}
$ gpg2 --edit-key saminiir@gmail.com
gpg> list
gpg> key 2
gpg> revkey
{% endhighlight %}

Again, do not delete revoked keys. They are still useful to decrypt data previously encrypted with the old key.

## Distributing an encryption subkey

For distributed usage, a subkey can be created for each usage purpose. 

For example, to encrypt and decrypt your password manager files both in your phone and laptop, transfer the generated encryption subkey to the phone like so:

{% highlight bash %}

# generate a strong random password
$ gpg2 --armor --gen-random 1 20
$ gpg2 --armor --export-secret-keys $SUBKEY_ID | gpg2 --armor --symmetric --output encsubkey.sec.asc
{% endhighlight %}

After that, transfer the `encsubkey.sec.asc` via a secure mechanism, e.g. a flash storage, to the new device.

Never transfer private keys through the internet, and strive for isolated environments where the transfer is done.

## Synchronizing the keyring between computers

Synchronizing a GPG keyring between computers may be desired to propagate information about imported key IDs[^gpg-synchronizing].

On whichever computer, export the keyring and put it on the encrypted USB drive:

{% highlight bash %}
$ gpg --export-secret-keys > my-secret-keyring.gpg
$ gpg --export-options export-local-sigs --export > my-public-keyring.gpg
{% endhighlight %}

Now, on the other computer, import (merge) the keyring:

{% highlight bash %}
 
$ gpg --import my-secret-keyring.gpg
$ gpg --import-options import-local-sigs --import my-public-keyring.gpg

{% endhighlight %}

Repeat this process the other way around, if needed.

{% include twitter.html %}

# Notes

[^1]:<https://en.wikipedia.org/wiki/Public-key_cryptography>
[^2]:<https://en.wikipedia.org/wiki/GNU_Privacy_Guard>
[^3]:<https://tools.ietf.org/html/rfc4880>
[^4]:<https://en.wikipedia.org/wiki/Web_of_trust>
[^5]:<https://en.wikipedia.org/wiki/X.509>
[^6]:<https://www.gnupg.org/faq/gnupg-faq.html#no_default_of_rsa4096>
[^7]:<https://www.gnupg.org/gph/en/manual/c481.html>
[^8]:<https://www.debian-administration.org/users/dkg/weblog/97>
[^9]:<https://en.wikipedia.org/wiki/Pretty_Good_Privacy#Criminal_investigation>
[^10]:<https://theintercept.com/2015/03/26/passphrases-can-memorize-attackers-cant-guess/>
[^11]:<https://futureboy.us/pgp.html#PerfectKeypair>
[^12]:<http://davidsoergel.com/posts/thoughts-on-gpg-key-management>
[^13]:<https://wiki.debian.org/Subkeys>
[^gpg-synchronizing]:<https://lists.gnupg.org/pipermail/gnupg-users/2011-May/041763.html>

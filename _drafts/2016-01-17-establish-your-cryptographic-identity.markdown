---
layout: post
title:  "Establish your online identity using GnuPG"
date:   2016-01-17 08:00:00
categories: cryptography
permalink: establish-cryptographic-identity-using-gnupg
---

I have a confession to make.

I do not have an online identity. I have Twitter, Facebook (yeah I know), this homepage, but still, my online presence is undermined by the fact that I or others cannot verify my identity.

The remedy to this is public-key cryptography[^1].

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# About PGP

Using GPG[^2], the OpenPGP-compliant[^3] cryptographic software suite, you are able to prove your identity online and that of others. The principle behind this is called the web of trust[^4], where users of OpenPGP-compatible software sign each others public signatures (keys) after having verified their identity. This is in contrast to X.509[^5], where a Certificate Authority, CA, is the sole party establishing the authenticity of users.

So in a way, PGP adheres more to the open source philosophy of decentralized, bazaar-like functions. Plus, the story of PGP is pretty bad-ass[^9].

In this tutorial, we'll refer to the OpenPGP specification as "PGP" and "GPG" as the free software implementation by the GNU Project.

TODO: Downsides of both OpenPGP and X.509 http://openpgp.org/technical/whybetter.shtml

# Using GPG

Creating and managing PGP keys is not a straightforward matter. Many approaches exist and if you are a whistleblower, this tutorial probably does not meet your security standards. In fact, if there's a chance you'll be captured, tortured, or killed for the information you'll encrypt, stop reading this tutorial and pray for the best. You obviously don't know what you're doing.

For others, I think this tutorial serves as a suitable middle ground for the pragmatic daily usage of PGP.

TODO: Difference between C, S, E

TODO: Explain actual impact of Web of Trust

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

Select the default RSA and RSA for both making signatures and encryption:

{% highlight bash %}
Pleasese select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
{% endhighlight %}

Next you will be asked to input the size for the RSA keys. We'll use 4096 bits for the master key, since it is only meant for signing other keys. The subkeys we'll use are used for encryption and decryption, so they are better off with a 2048 bit key size[^6].

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

Input your real name and email. Do not, however, input a comment when prompted[^8]:

{% highlight bash %}
GnuPG needs to construct a user ID to identify your key.

Real name: Sami Niiranen
Email address: saminiir@gmail.com
Comment:
You selected this USER-ID:
    "Sami Niiranen <saminiir@gmail.com>"
{% endhighlight %}

Next up, you have to come up with a passphrase for your master key. The Intercept has an insighftul article[^10] on the problems of passphrases and how to create robust ones (with Diceware).

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
$ gpg2 --edit-key $masterkeyid
$ gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
Your selection? 4

{% endhighlight %}

Choose the `RSA (sign only)` option for generating a signing key only. Use the default 2048-bit keysize if you have no reason to use a larger one. Choose an expiration date. Input a passphrase.

Repeat the procedure, but this time, create a RSA (encrypt only) -key.

As a final step, save the keys you've made:

{% highlight bash %}
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

There are endless possibilities how to store your master keypair, from paper printouts in a safe to armed guards in your basement. Because I haven't yet obtained any interesting information to whistleblow, I chose the mundane approach of backing up the master keypair to a USB drive:

{% highlight bash %}
$ gpg2 --export-secret-keys --armor $keyid > privkey
$ gpg2 --export-keys --armor $keyid > pubkey
$ gpg2 --export-secret-subkeys $keyid > subkeys
$ gpg2 --delete-secret-key $keyid
$ gpg2 --delete-key $keyid
$ gpg2 --import subkeys
$ shred subkeys
{% endhighlight %}

You can verify that the master secret key should not be in the keyring by detecting `sec#` when listing the secret keys.

Now, be sure to evacuate your master keypair `privkey` and `pubkey` with your chosen method to a safe location. I'm storing it in an encrypted USB drive.

## Sending the public key to a key server

Next, you'll want your public key to be actually obtainable for other people. Keyservers are a good fit for this, since the gpg tool can be used to query them.

{% highlight bash %}
$ gpg2 --keyserver hkp://pool.sks-keyservers.net --send-key $masterkeyid
$ gpg2 --keyserver hkp://pool.sks-keyservers.net --search-keys $youremail
{% endhighlight %}

This sends your public key to the popular sks keyserver pool. Afterwards you and anybody else can find your PGP identity.

# Scenarios

## New device, new subkey
## Signing someone's key
## Extending lifetime of a key
## Revoking a subkey

# Sources

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

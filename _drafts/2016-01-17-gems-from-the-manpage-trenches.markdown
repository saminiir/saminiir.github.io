---
layout: post
title:  "Gems from the man page trenches"
date:   2016-01-17 08:00:00
categories: cryptography
permalink: gems-from-man-page-trenches/
---

I'm a big fan of UNIX man pages[^1]. They are almost without exception well-written and concise, and the intuition you gain for particular topics when you browse through the man pages is well worth the effort. In contrast, some developers shoot straight to googling something like "How to print date with format in Linux". 

I argue this is counter-intuitive - one ends up with poor StackOverflow answers and the intended use of applications is often left out. Even the "I don't have time for reading through pages of man pages" argument does not hold, since that just stems out of lack of routine with the UNIX environment. 

Without further ado, here's some funny and interesting parts I've discovered in man pages:

The manpage for gnupg has 
{% highlight bash %}
--gen-prime mode bits
        Use the source, Luke :-). The output format is still subject to change.
{% endhighlight %}

crontab has
{% highlight bash %}
# run at 10 pm on weekdays, annoy Joe
0 22 * * 1-5    mail -s "It's 10pm" joe%Joe,%%Where are your kids?%
{% endhighlight %}

xxd :
{% highlight bash %}
WARNINGS
       The  tools weirdness matches its creators brain.  Use entirely at your own risk. Copy files. Trace it. Become
       a wizard.
{% endhighlight %}

This one is lengthier, but underlines the pragmatism of the UNIX philosophy. From `man git-tag`:

{% highlight bash %}
What should you do when you tag a wrong commit and you would want to re-tag?

If you never pushed anything out, just re-tag it. Use "-f" to replace the old one. And you’re done.

But if you have pushed things out (or others could just read your repository directly), then others will
have already seen the old tag. In that case you can do one of two things:

1. The sane thing. Just admit you screwed up, and use a different name. Others have already seen one
   tag-name, and if you keep the same name, you may be in the situation that two people both have "version
   X", but they actually have different "X"'s. So just call it "X.1" and be done with it.

2. The insane thing. You really want to call the new version "X" too, even though others have already seen
   the old one. So just use git tag -f again, as if you hadn’t already published the old one.
{% endhighlight %}

Please share your favorite ones!

[^1]:<https://en.wikipedia.org/wiki/Man_page>

# TRYHACKME — PACKED LIGHT

This room is a part of THM’s HackerHolidays 2026.

---

### TRYHACKME — PACKED LIGHT

This room is a part of THM’s HackerHolidays 2026.

![](https://cdn-images-1.medium.com/max/800/1*gLN528lNhG52-rRzdPDq-A.png)

Tiny packets. Odd hours. Suspiciously regular. Someone’s smuggling out the data equivalent of a hotel towel every night, folded neatly inside traffic that looks ordinary until you decode it.

![](https://cdn-images-1.medium.com/max/800/1*io6Kl9ACICbV0pXzHGh90w.png)

So the room is based on wireshark packet analysis, according to the story given to us we just have to focus on port 8080.

![](https://cdn-images-1.medium.com/max/800/1*zHY4yYPgEQwTQzFk6P8h4Q.png)

There is something /temp/updates.py,

Follow its stream.

![](https://cdn-images-1.medium.com/max/800/1*S0U7UFWDnF13WiwhrxTG-g.png)

So what this script basically does is

First generate a key by concatenating the 2 parts “H0t3lSt@ff0NlyK3epS3cr3t!

Then XOR on the letter entered

Then Base64 encode it

And sends it as a cookie (each request only 1 letter)

The first cookie is

![](https://cdn-images-1.medium.com/max/800/1*rEmWeMD_aZs7CAkTyNI8Cw.png)

HA== which wen first base64 decoded then XOR decoded gives:-

![](https://cdn-images-1.medium.com/max/800/1*oc1XtXerLhJeTghSEDopOw.png)

‘T’

Tryhackme’s flags always start with THM{flag}

Which confirms that we are on the right track.

After getting all cookie values and performing this on them all at once,

we get the flag!

![](https://cdn-images-1.medium.com/max/800/1*W9KBC0b3HSp2Q2z6WXcvkw.png)

Thanks for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [July 30, 2026](https://medium.com/p/f67adeb45e9f).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-packed-light-f67adeb45e9f)

Exported from [Medium](https://medium.com) on August 8, 2026.

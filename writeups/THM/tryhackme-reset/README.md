# TRYHACKME — RESET

This challenge simulates a cyber-attack scenario where you must exploit an Active Directory environment.

---

### TRYHACKME — RESET

![](https://cdn-images-1.medium.com/max/800/1*_k2O2VFjLdfIz1dY29CzwA.png)

This challenge simulates a cyber-attack scenario where you must exploit an Active Directory environment.

Reset is a hard level room on TryHackMe which focuses on Active Directory exploitation.

Starting with a basic nmap scan :-

![](https://cdn-images-1.medium.com/max/800/1*Br8FuZxA-croftYSsB1srA.png)

Add HAYSTACK haystack.thm.corp HAYSTACK.thm.corp thm.corp to our /etc/hosts file.

Trying SMB Null session :-

![](https://cdn-images-1.medium.com/max/800/1*MEGvMDKL0rcJzu63Ecd_dA.png)

Data share can be viewed,

![](https://cdn-images-1.medium.com/max/800/1*KWCkkDQI6xSETAKi2wM5pQ.png)

Reading the files that i got from smb :-

![](https://cdn-images-1.medium.com/max/800/1*nIEG8RiLzPYD5F9bSCTqGQ.png)

Anddd we have a password, but we still need a username.

Let’s try using netexec to perform rid-bruteforcing and get some usernames.

![](https://cdn-images-1.medium.com/max/800/1*boOkV-11KMT3O_d9NtEKlg.png)

Now that we have some usernames we can try password spraying.

![](https://cdn-images-1.medium.com/max/800/1*V0nx-1qUcIhbAN2En1agyw.png)

Although we got this user, i couldn’t get it work on anything such as smb.

I tried checking if ASREP Roasting was possible :-

![](https://cdn-images-1.medium.com/max/800/1*u7whITrFvECcrKFkzcXGYg.png)

Now lets crack this :-

![](https://cdn-images-1.medium.com/max/800/1*iUy8Gyz0ebzCDTCxNmrcPQ.png)![](https://cdn-images-1.medium.com/max/800/1*FHe-lNBOJoDhvLlQuxMesA.png)

Anddd we got a password!

Let’s try running bloodhound :-

![](https://cdn-images-1.medium.com/max/800/1*qCtr6xL_GqpfSZhlpL_Sfg.png)![](https://cdn-images-1.medium.com/max/800/1*nz88Kd-m3gp8GLketHL9wQ.png)

Sooo, we have a chain

First we make use of GenericAll to reset password of SHAWNA

Then we make use of ForceChangePassword 2 times to get to DARLA.

![](https://cdn-images-1.medium.com/max/800/1*HaTHrd-poU1zAd7H11X_fA.png)

Now Darla has Constrained Delegation on the DC, which means we can make use of this to impersonate admin and get his ticket.

![](https://cdn-images-1.medium.com/max/800/1*AoJp-dcdHBVsQHOasv7ROw.png)![](https://cdn-images-1.medium.com/max/800/1*BLsRNnANIqOeufpINl8lEg.png)

Psexec and evil-winrm weren’t working so i tried wmiexec :-

![](https://cdn-images-1.medium.com/max/800/1*hK0WDvzS7Z4jIJGYEAwm7g.png)

USER FLAG :-

![](https://cdn-images-1.medium.com/max/800/1*OZTi4ZCDkzeQd-b7_xTPRg.png)

ROOT FLAG :-

![](https://cdn-images-1.medium.com/max/800/1*3kaQybhc300pAQbvZZA-XA.png)

Thanks for reading!

Note :- I switched between THM’s attackbox and my personal machine quite a lot while doing this room.

By [Lightningfst](https://medium.com/@lightningfst8) on [August 2, 2026](https://medium.com/p/85935ee844e7).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-reset-85935ee844e7)

Exported from [Medium](https://medium.com) on August 8, 2026.

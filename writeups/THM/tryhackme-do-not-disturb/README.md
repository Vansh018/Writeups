# TRYHACKME — DO NOT DISTURB

This room is a part of THM’s HackerHolidays 2026.

---

### TRYHACKME — DO NOT DISTURB

![](https://cdn-images-1.medium.com/max/800/1*CBEBY7ymwS36nwOUTj7rSQ.png)

This room is a part of THM’s HackerHolidays 2026.

Sign’s on the door. Room’s active. You have access you were never given, and so does he.

![](https://cdn-images-1.medium.com/max/800/1*cretcLlsltgG5L8-8HTxog.png)![](https://cdn-images-1.medium.com/max/800/1*TdDcmUhBwSHJIB1G_cTlEA.png)

Soo, i wasn’t able to make out much from the room’s description.

Visiting the web page, i found a login page,

![](https://cdn-images-1.medium.com/max/800/1*aO0wYSWIPfy8HNLEo3z5pA.png)

Also performing directory enumeration gave me a /staff endpoint.

![](https://cdn-images-1.medium.com/max/800/1*YKyC0dW_XI48hz4ELRSJHw.png)

Hmmm, i cant access it like this.

Using Wappalyzer, it was confirmed that this was a node/Express based webserver.

I tried some authentication bypass tricks from hacktricks but they didn’t work.

I also tried running sqlmap on the login page but found nothing.

I was lost for a while, then i decided to try NoSQL Injection.

![](https://cdn-images-1.medium.com/max/800/1*52NQtEiyWAIg5WsXqGatzg.png)

anddd, i successfully bypass the login page.

![](https://cdn-images-1.medium.com/max/800/1*igIIux4o8xU4uGuFMjKVwA.png)

Let’s try accessing the /staff page.

![](https://cdn-images-1.medium.com/max/800/1*4Njh0MoR1QDHGTmw8GCyAA.png)

The preview button and the guest thing straight up tell me that this is SSTI related.

![](https://cdn-images-1.medium.com/max/800/1*WmyiwAVxseEq4a7doQ3p7Q.png)

SSTI confirmed.

Let’s get a reverse shell.

I used the following payload :-

![](https://cdn-images-1.medium.com/max/800/1*gIcVjfFD4nHliXNMT9vbtA.png)![](https://cdn-images-1.medium.com/max/800/1*UeKY8mIu5Jqojyq6qi8VjQ.png)

andd i got a shell.

![](https://cdn-images-1.medium.com/max/800/1*lnR7qYCgFxs2ASas8bcJBA.png)

hmmmm interesting!

Let’s get the user flag first from the poolside user’s home directory.

THM{\*\*\*\*\_\*\*\*\*\*\*\*\_\*\*\*\*\*\*\*\*}

Now the privesc part,

I ran linpeas and saw that node was running on local port 9229.

P.S. :- I took a very very long way to get root, there were easier methods too.

After reading the files in the telemetry folder we found earlier, i took help from claude to create a script which would achieve RCE :-

![](https://cdn-images-1.medium.com/max/800/1*nhZ-0U8WWiwyb_Clx6GX8Q.png)

Running this script, i got a reverse shell as pipelinesvc.

![](https://cdn-images-1.medium.com/max/800/1*bVYmuqhWxpkGsf3KDxsHzQ.png)

Also, we are part of ‘disk’ group.

I looked on the internet for privelege escalation by disk group and found some commands.

![](https://cdn-images-1.medium.com/max/800/1*VJcmAo5W1zvhqDHPx5K18Q.png)

And we goot the root flag!

Instead of trying to get a root shell, i just read the root.txt flag straight away.

Thanks for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [August 3, 2026](https://medium.com/p/0f77c3edcb0d).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-do-not-disturb-0f77c3edcb0d)

Exported from [Medium](https://medium.com) on August 8, 2026.

# TRYHACKME — INFINITY POOL

This room is a part of THM’s HackerHolidays 2026.

---

### TRYHACKME — INFINITY POOL

This room is a part of THM’s HackerHolidays 2026.

![](https://cdn-images-1.medium.com/max/800/0*J56CAddB87W20r3i.png)

No visible edge. You trace the network to the horizon and find three systems nobody told you about on the other side.

Performing a basic nmap scan showed port 80 open,

Directory busting found /status.

![](https://cdn-images-1.medium.com/max/800/0*FKpLH4eHdC62cC-C.png)

Classic Command Injection scenario.

![](https://cdn-images-1.medium.com/max/800/0*3MGDfsWl6xH4MR6L.png)

Command injection confirmed!

Getting a reverse shell :-

![](https://cdn-images-1.medium.com/max/800/0*r9mSF6w5jO5ySuNm.png)![](https://cdn-images-1.medium.com/max/800/0*ElCm07yj4da4hh87.png)

Now, we can get the user flag.

![](https://cdn-images-1.medium.com/max/800/0*8VG5inCZx-ARtRYv.png)

After doing some local enumeration such as checking local ports, i found port 8080, 9000, 3000 open.

![](https://cdn-images-1.medium.com/max/800/0*MtHEeg1QA_jI6JrS.png)

And found this at port 3000.

I created ssh key pair to perform ssh local port forwarding to make it easier accessing these webpages.

![](https://cdn-images-1.medium.com/max/800/0*zz2pv5CNUjH0-kDR.png)![](https://cdn-images-1.medium.com/max/800/0*sxiRMrN3BBRkEzDK.png)

Found this in port 9000/health.

![](https://cdn-images-1.medium.com/max/800/0*J00KnC6KJMFn_5w5.png)

Andd found this at port 8080/admin/config.php

Clicking on the user control panel it asks for user and pass and takes us to /ucp.

We can use the user and password we found earlier here.

Now, Click on the top right + to create a dashboard, after that click on the left side + icon and select VOICEMAIL, automation key is present here.

![](https://cdn-images-1.medium.com/max/800/0*fsMb9GuXSqsWmE3R.png)

Now we can make a call to port 3000/jobs/export.

![](https://cdn-images-1.medium.com/max/800/0*n-9PeSXgvtDPDwmu.png)

Andd we can see that the app inserts the data from report parameter straight into the command line, we can make use of command injection along with # to comment out rest of the command.

![](https://cdn-images-1.medium.com/max/800/0*KdfKDije_UW78QnI.png)

Getting a revshell :-

![](https://cdn-images-1.medium.com/max/800/0*Sj4uoaR52h_XdJaM.png)![](https://cdn-images-1.medium.com/max/800/0*l5pjJMcQWIBK43Ov.png)

Get the root flag!

Thanks for reading.

By [Lightningfst](https://medium.com/@lightningfst8) on [August 7, 2026](https://medium.com/p/9ed848167c02).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-infinity-pool-9ed848167c02)

Exported from [Medium](https://medium.com) on August 8, 2026.

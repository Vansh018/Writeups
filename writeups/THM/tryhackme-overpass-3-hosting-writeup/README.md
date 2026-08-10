# TRYHACKME — OVERPASS 3:HOSTING WRITEUP

Overpass 3-Hosting is a medium level room on tryhackme, it includes basic directory enumeration, encrypted files, using ftp to get shell…

---

### TRYHACKME — OVERPASS 3:HOSTING WRITEUP

![](https://cdn-images-1.medium.com/max/800/1*V3VTr5MgamhZqc0B_mcTJg.png)

Overpass 3-Hosting is a medium level room on tryhackme, it includes basic directory enumeration, encrypted files, using ftp to get shell, insecure nfs to privesc.

performing a nmap scan showed port 21,22,80 open.

![](https://cdn-images-1.medium.com/max/800/1*8d-yu_ATEWk4lANDAzw-4g.png)

found this running on port 80 and decided to perform directory busting.

![](https://cdn-images-1.medium.com/max/800/1*_KM0W5UlYnO8Qz4tyVfPGA.png)![](https://cdn-images-1.medium.com/max/800/1*_Akas0JFo_T0tv93Krxamw.png)

going to the /backups directory we can see a backup.zip file, after downloading and extracing it we get a pgp key and an encrypted file

![](https://cdn-images-1.medium.com/max/800/1*4Uw1DOBINv2kMHsNKy_p0Q.png)

we can use the key to decrypt the file

![](https://cdn-images-1.medium.com/max/800/1*eyanEly5OjhSXGuXLnCjrg.png)

in the xlsx file we get some username and passwords

![](https://cdn-images-1.medium.com/max/800/1*EaAHZOpKfnL1tgDD8lJWcg.png)

using the password for paradox, we can login into ftp

![](https://cdn-images-1.medium.com/max/800/1*KQ7R_KmG_bMgu10iIpGOew.png)

here we can see that ftp is directly linked with the web app running, we can upload a php shell using ftp and run it by visiting the link.

![](https://cdn-images-1.medium.com/max/800/1*-TjA9L3RQ91BXj0NzWVavQ.png)![](https://cdn-images-1.medium.com/max/800/1*jwAtPnP7G4gHPw1ImZszUg.png)

now we can search for the web flag

![](https://cdn-images-1.medium.com/max/800/1*VD1NE660hBtMdV59lIfsXg.png)![](https://cdn-images-1.medium.com/max/800/1*gqlESjPtEiiBMii8BQh3xQ.png)

web flag :- thm{0ae72f7870c3687129f7a824194be09d}

now, first we change user to paradox as we have the password for him already.

now we create ssh keys for paradox and use them to stabilize our shell.

![](https://cdn-images-1.medium.com/max/800/1*T_pfZlZFpDTVCM9nea5w6g.png)

paste the contents of paradox.pub to /home/paradox/.ssh/authorized\_keys

then login using the private key.

for the privesc part, we can put linpeas to the target using ftp and then use it.

Linpeas scan shows that the user ‘james’ `james` has an insecure NFS configuration.

[i] <https://book.hacktricks.xyz/linux-unix/privilege-escalation/nfs-no_root_squash-misconfiguration-pe>  
/home/james \*(rw,fsid=0,sync,no\_root\_squash,insecure)

nfs is running locally so first we check on which port its running on,

![](https://cdn-images-1.medium.com/max/800/1*5OeNwwGmAhmPzEpIbpASTA.png)

its running on port 2049

we can use ssh to port forward it to our machine.

and now we mount nfs

![](https://cdn-images-1.medium.com/max/800/1*s59T6mUZDOWCNMt4xhvC-g.png)![](https://cdn-images-1.medium.com/max/800/1*BDRWb_GyHtFU9if4PrZe9Q.png)

user flag :- thm{3693fc86661faa21f16ac9508a43e1ae}

now we can use the same key from paradox here

PRIVILEGE ESCALATION:-

what we can do here is copy bash for user james to the nfs shared location and then using our machine apply suid to it.

```
#In the mounted dir, as root user  
cp /bin/bash ./
```

root flag:- thm{a4f6adb70371a4bceb32988417456c44}

thanks for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [June 21, 2026](https://medium.com/p/fc500f58e30f).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-overpass-3-hosting-writeup-fc500f58e30f)

Exported from [Medium](https://medium.com) on August 8, 2026.

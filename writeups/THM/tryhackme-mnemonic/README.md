# TRYHACKME — MNEMONIC

Mnemonic is a medium level room on tryhackme.

---

### TRYHACKME — MNEMONIC

![](https://cdn-images-1.medium.com/max/800/1*TVYn1vR38u2WVDMucQHKNQ.png)

Mnemonic is a medium level room on tryhackme.

It’s description :- I hope you have fun.

Starting with a basic nmap scan,

![](https://cdn-images-1.medium.com/max/800/1*JIFmqXAkZYJB8YUiRphc0Q.png)

we find 3 ports are open,

21 — ftp

80 — http

1337 — ssh

I couldn’t think of anything to do with ftp as it didn’t allow anonymous logins.

I visited the web page and checked the robots.txt

![](https://cdn-images-1.medium.com/max/800/1*Zk4P8ScTOVKMd_thFB9NoA.png)

aaah, found a hidden directory,

visiting the directory we dont see anything, but we can perform directory enumeration.

![](https://cdn-images-1.medium.com/max/800/1*T8a7MSA08bN9t92zvpiMjQ.png)

interesting, an admin page as well as a backups directory,

the admin page doesn’t have anything useful so let’s perform more directory enumeration on the backups directory.

![](https://cdn-images-1.medium.com/max/800/1*6o4PPy7PI8teGa9lkxZisA.png)

anddd, we find a backups.zip file.

lets download and try unzipping it.

The file is password protected, we can use zip2john.

![](https://cdn-images-1.medium.com/max/800/1*UJNYSUPvZVjcrtfcUhWm_Q.png)

inside the zip, we find a note.txt file.

![](https://cdn-images-1.medium.com/max/800/1*YLdk55utjq7mxpNIVoaYRQ.png)

so this gives us the ftp username, now hydra can be used to find the password.

hydra -l ftpuser -P /usr/share/wordlists/rockyou.txt 10.49.160.43 ftp

![](https://cdn-images-1.medium.com/max/800/1*pxPg2Nwm1ZeFgbjX1l4riA.png)

in one of the directories we can find id\_rsa and a not.txt

![](https://cdn-images-1.medium.com/max/800/1*-mzNim7eYl35xuk1KJLlqg.png)

so we have a username and a ssh key, but its locked behind a password, use ssh2john.

![](https://cdn-images-1.medium.com/max/800/1*Wdxt1HSTZErHbn12sy0QQA.png)

use it login as james through ssh.

![](https://cdn-images-1.medium.com/max/800/1*zM8-UHAUZKOzS8zsedtQCA.png)![](https://cdn-images-1.medium.com/max/800/1*yMv6VjhSUZ0chalndUyI0Q.png)

and in the 6450.txt we find this,

![](https://cdn-images-1.medium.com/max/800/1*BYyOzr4W4cYUT2OPgMPzaw.png)

i searched for mnemonic encryption on google, and in the github repository i found a tool to encrpyt and decrypt images, but we don’t have an image yet.

we can’t cd out.

maybe try ls

![](https://cdn-images-1.medium.com/max/800/1*FqnQUCCHMqicpJPMhE1qog.png)

we see 2 b64 encoded strings,

one is a flag and the other has the img link, download that img and use the tool to decrypt it.

and we get the password for condor.

use ssh to login as condor.

![](https://cdn-images-1.medium.com/max/800/1*IqXZlnNE_Yr5b0kXgpKRnQ.png)

we see we can use sudo on a python script.

cat the script and you can see that, if we choose 0

it takes unsanitized input from the user

if select == 0:   
 time.sleep(1)  
 ex = str(input(“are you sure you want to quit ? yes : “))  
   
 if ex == “.”:  
 print(os.system(input(“\nRunning….”)))  
 if ex == “yes “ or “y”:  
 sys.exit()

we use this to our advantage

![](https://cdn-images-1.medium.com/max/800/1*0cW68CDTNLhTXjoqoOMSGQ.png)![](https://cdn-images-1.medium.com/max/800/1*m1fH9t79M29tBKALu-WAxQ.png)![](https://cdn-images-1.medium.com/max/800/1*Qp87mhchsXW9uvdGZb47Yg.png)

and boom, we are root, now just get the root flag from /root/root.

![](https://cdn-images-1.medium.com/max/800/1*FJ2lwZ7Bv3WfiBI5H49mCg.png)

Thanks for Reading!

Ps.. generate md5 hash of the contents of the flag for the actual flag (root).

By [Lightningfst](https://medium.com/@lightningfst8) on [July 11, 2026](https://medium.com/p/b9b507e169d0).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-mnemonic-b9b507e169d0)

Exported from [Medium](https://medium.com) on August 8, 2026.

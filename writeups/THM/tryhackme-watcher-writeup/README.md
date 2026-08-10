# TRYHACKME — WATCHER WRITEUP

watcher is a medium-hard difficulty room on tryhackme, it includes basic enumeration such as checking robots.txt, lfi 2 rce, privelege…

---

### TRYHACKME — WATCHER WRITEUP

![](https://cdn-images-1.medium.com/max/800/1*uMcdlK-IJoUBAbs7C5_dTA.png)

watcher is a medium-hard difficulty room on tryhackme, it includes basic enumeration such as checking robots.txt, lfi 2 rce, privelege escalation using sudo, cronjobs, stored db credentials.

starting with basic enumeration, reading the robots.txt file reveals flag 1 and a secret directory.

![](https://cdn-images-1.medium.com/max/800/1*DG_G0d1P0O0Fh6aHY0BMqQ.png)

flag1 :- FLAG{robots\_dot\_text\_what\_is\_next}

trying to access the secret directory gives this

![](https://cdn-images-1.medium.com/max/800/1*bdERDp59ld_F2TTIvYnBAQ.png)

coming back to the home page, we can see a lot of posts present, reading the source we can find the link to the posts and open one of them

![](https://cdn-images-1.medium.com/max/800/1*vVKvwfoJ6dx3iaa1e9gL7A.png)![](https://cdn-images-1.medium.com/max/800/1*pjTdrg1I6TTot44-1ZSrZA.png)

in the post’s url we can see that post parameter is responsible for loading the pages

given that the files are being loaded directly, we can try reading the secret directory using this

![](https://cdn-images-1.medium.com/max/800/1*ZcyUroSkyx_e4OSiyJ_DSw.png)

and yepp, we get the ftp creds.

logging into ftp

![](https://cdn-images-1.medium.com/max/800/1*-tCKzxvR0PkWZEdIaQ84XQ.png)

flag 2 :- FLAG{ftp\_you\_and\_me}

now coming back to the lfi i try reading some common files

![](https://cdn-images-1.medium.com/max/800/1*Yn1ii6AfAbRWtdHu9QJyfw.png)

considering we have lfi and ftp, we can try uploading a php shell using ftp and then call it with lfi.

![](https://cdn-images-1.medium.com/max/800/1*9FQM2CJi9bkQw3iUSUmfag.png)

start listener and call the file using lfi

![](https://cdn-images-1.medium.com/max/800/1*r3dC6eMHGi3tX-ZPJy8aOw.png)![](https://cdn-images-1.medium.com/max/800/1*hpPHNybVKlwsfj--1a0q8A.png)

and we have shell, looking for flag 3 in the web root directory

![](https://cdn-images-1.medium.com/max/800/1*yVtSJbJtPWB9VOL-K9CCXw.png)

flag 3 :- FLAG{lfi\_what\_a\_guy}

now we look for priv esc vectors,

![](https://cdn-images-1.medium.com/max/800/1*6blHgiMhRq24RAg_q3qzrQ.png)

cant believe we got it so easily,

![](https://cdn-images-1.medium.com/max/800/1*BIgJ9N4D7eugLEDqDqx6hw.png)

flag 4 :- FLAG{chad\_lifestyle}

![](https://cdn-images-1.medium.com/max/800/1*TbKOs78VDeQlwJwsRbMpsw.png)

here we can see a note present in toby’s home directory which hints at cronjob being ran, also there is a jobs folder present too.

![](https://cdn-images-1.medium.com/max/800/1*JXdAfmXK8qu8yMqb4CQFBA.png)

we can edit the cow.sh and add a reverse shell to it

![](https://cdn-images-1.medium.com/max/800/1*7wNGoSAqdVUekppWR4lTOA.png)

after some time we should get a shell

![](https://cdn-images-1.medium.com/max/800/1*_g38Rmz6NE3Chq9VWml7bg.png)

flag 5:-FLAG{live\_by\_the\_cow\_die\_by\_the\_cow}

![](https://cdn-images-1.medium.com/max/800/1*imTtHFKFv2gJIZcN_WXGCA.png)

we again find a note.txt and this time it mentions a python script we can run as will

![](https://cdn-images-1.medium.com/max/800/1*bqvQrGiCLTxISJlSuYQP_g.png)![](https://cdn-images-1.medium.com/max/800/1*FS3O2SEFqiJXUrNd9ZxqEg.png)

considering that mat has ownership on cmd.py and will\_script.py calls cmd.py we can add a shell into cmd.py

![](https://cdn-images-1.medium.com/max/800/1*SPn-1EnS6ipdvy4H3UpNRQ.png)![](https://cdn-images-1.medium.com/max/800/1*Hv7sKWmjpxLOT9tx2cGfWQ.png)![](https://cdn-images-1.medium.com/max/800/1*Dz280xWX4-0OGVQs0BzdHA.png)

start a listener and run the script we should get a shell

![](https://cdn-images-1.medium.com/max/800/1*3cLs3Zi1O5M3yz1kVTIvRA.png)![](https://cdn-images-1.medium.com/max/800/1*H-J_fZn113PkTe1Cto6ctA.png)

flag 6:- FLAG{but\_i\_thought\_my\_script\_was\_secure}

![](https://cdn-images-1.medium.com/max/800/1*zjgDzgFaTGfvmZZsqtVChg.png)

now we perform general privelege escalation enumeration.

![](https://cdn-images-1.medium.com/max/800/1*nDZsUs50w-R2rqEY7ZJKRg.png)

looks like will is part of adm group, which means we can read log files

although after searching alot i couldnt find something in the log files so i decided to check other common folders and found this

![](https://cdn-images-1.medium.com/max/800/1*3A3pnF-cTV_iHwteKmyJmA.png)![](https://cdn-images-1.medium.com/max/800/1*GhFPhuwemPARHWezRYMxwA.png)

decoding this base64 we get a ssh key (for root, as there are no other users left) , we use it to get root

![](https://cdn-images-1.medium.com/max/800/1*OfCgoIgr93qA0OgFzZImjw.png)![](https://cdn-images-1.medium.com/max/800/1*3ZSPRWMLHqZZZXte-NIwkA.png)

flag 7:- FLAG{who\_watches\_the\_watchers}

Thank you for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [June 20, 2026](https://medium.com/p/43f8f7ea0486).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-watcher-writeup-43f8f7ea0486)

Exported from [Medium](https://medium.com) on August 8, 2026.

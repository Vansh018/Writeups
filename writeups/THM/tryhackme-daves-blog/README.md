# TRYHACKME — DAVE’S BLOG

My friend Dave made his own blog!

---

### TRYHACKME — DAVE’S BLOG

![](https://cdn-images-1.medium.com/max/800/1*4364bN2LbdDkdWdjoy7GUQ.png)

My friend Dave made his own blog!

Starting with a basic nmap scan:-

![](https://cdn-images-1.medium.com/max/800/1*WeTdrNjwZUiMGoHSTbWcdA.png)

Open ports :-

Port 22

Port 80

Port 3000

Port 8989

There was the same thing running on port 3000 and port 80,

![](https://cdn-images-1.medium.com/max/800/1*hzQ9qATNm_3Oox1zmnMTYQ.png)

from robots.txt we see a hidden directry /admin.

![](https://cdn-images-1.medium.com/max/800/1*BQZxLbI6qGoeEzCvTmKVDg.png)

We have a login form here, and also the home page suggests that nosql is running which means that we can try authentication bypass here.

Also looking at the source of the /admin page gives us some idea about the payload that we should use.

![](https://cdn-images-1.medium.com/max/800/1*8Rh3V-1pW5kUqeNNE1xLkg.png)

Using this payload, the form is bypassed and we get a jwt cookie which when decoded gives us the first flag.

![](https://cdn-images-1.medium.com/max/800/1*mmQ9lWuDpfA82z0F7xAxug.png)

Setting this jwt as our cookie in the browser, we can now access the admin page.

![](https://cdn-images-1.medium.com/max/800/1*qCV4r4rCFFgnnh7lhENK1Q.png)

The exec button could be hinting that the js exec function is being used, which can be used to get a reverse shell using the following payload,

[**Tryhackme-Dave'sBlog - Pastebin.com**  
*Pastebin.com is the number one paste tool since 2002. Pastebin is a website where you can store text online for a set…*pastebin.com](https://pastebin.com/PhV26hUp "https://pastebin.com/PhV26hUp")

Couldn’t paste the payload here, medium kept flagging it.

![](https://cdn-images-1.medium.com/max/800/1*t4HQTJQKvZrUSf_ctnvdtw.png)

Get the user flag.

Mongo db was also running, we get flag 3 from there.

![](https://cdn-images-1.medium.com/max/800/1*SpESqfMzfD4pBrL6LnnCwA.png)

Privilege escalation:-

We can run sudo -l and see that a binary uid\_checker can be run as root.

![](https://cdn-images-1.medium.com/max/800/1*xZkAfR--acndtPKKMRQdaA.png)

Now this part was very confusing for me as i am not that good in RE.

After taking some help from claude i was able to determine that the binary was vulnerable to a buffer overflow.

Took some time to curate the payload, but at the end i got it.

![](https://cdn-images-1.medium.com/max/800/1*oB2e3wwaGLn_N_3LGoFDOA.png)

Thanks for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [July 27, 2026](https://medium.com/p/61f4f3393d4b).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-daves-blog-61f4f3393d4b)

Exported from [Medium](https://medium.com) on August 8, 2026.

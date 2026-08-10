# TRYHACKME — COMPLIMENTARY

This room is a part of THM’s HackerHolidays 2026.

---

### TRYHACKME — COMPLIMENTARY

![](https://cdn-images-1.medium.com/max/800/1*bUdzjx2ASml-mOt5LXF0_g.png)

This room is a part of THM’s HackerHolidays 2026.

Install the free app and it hands your phone a set of cloud keys, the same set it hands everyone. They’re read-only, but read-only of every guest’s contacts, location, and passwords, not just Lambo’s. She gave consent. Technically.

![](https://cdn-images-1.medium.com/max/800/1*5dSwWLlr-NfO1CfXUytSRw.png)

This room was based on AWS and cloud, i had little to no experience in pentesting AWS/Cloud so i took some help from claude.

First of all i read the source of the page and found app.js.

![](https://cdn-images-1.medium.com/max/800/1*ShLCHsfYW6Wc5XsS9fu6Wg.png)![](https://cdn-images-1.medium.com/max/800/1*ud7IClZmqZDWk6kJupIvFA.png)

From here we get important stuff such as IDENTITY POOL ID, AWS\_REGION and TABLE NAME.

Also from the network tab in debug tools, i was able to find a set of keys and secrets that can be used to access the DB.

![](https://cdn-images-1.medium.com/max/800/1*oW6xUHUPmkvgcXAfI3-76w.png)

First of all setting the aws region:-

aws configure set region us-east-1

So now we can just set these keys that we get as variables in terminal.

![](https://cdn-images-1.medium.com/max/800/1*N1pwYvVZV1vNngI9waZTbQ.png)

Now we can check according to the secrets we supplied who are we,

![](https://cdn-images-1.medium.com/max/800/1*63IbAKMn8dCAXDdHUU41_Q.png)

Now we can return all the items from the table (claude helped here) :-

![](https://cdn-images-1.medium.com/max/800/1*2v4aerefmYv8GOjMVNsWMg.png)![](https://cdn-images-1.medium.com/max/800/1*blIwozWOWkOiS298vVzphg.png)

Andd we have the flag!

Thanks for reading.

By [Lightningfst](https://medium.com/@lightningfst8) on [July 29, 2026](https://medium.com/p/258a09d7d11e).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-complimentary-258a09d7d11e)

Exported from [Medium](https://medium.com) on August 8, 2026.

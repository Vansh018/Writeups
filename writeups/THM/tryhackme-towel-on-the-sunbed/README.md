# TRYHACKME — TOWEL ON THE SUNBED

This room is a part of THM’s HackerHolidays 2026.

---

### TRYHACKME — TOWEL ON THE SUNBED

![](https://cdn-images-1.medium.com/max/800/1*6BfrFF2xSfpUq2nZRnVUMg.png)

This room is a part of THM’s HackerHolidays 2026.

Ponzi set his towel down for one 24-hour reward claim. He came back to find the sunbed had been “claimed” three times over while he wasn’t looking.

![](https://cdn-images-1.medium.com/max/800/1*EmOq-y_D9zh2vAsndeo8GQ.png)

Seems like a room related to race conditions, lets see.

![](https://cdn-images-1.medium.com/max/800/1*A6zu9rB7_MekoKI1ejS6Ew.png)

Hmm, a login page.

I tried sqli but got nothing.

Let’s register as a new user.

![](https://cdn-images-1.medium.com/max/800/1*Px8nDvJegchXg5ujAqjYtA.png)

Now we can login into the account :-

![](https://cdn-images-1.medium.com/max/800/1*C7cuAnWd3DRMuieUYyIsBA.png)

Aaha, we have a claim reward option.

Let’s capture the request and see what we got.

![](https://cdn-images-1.medium.com/max/800/1*LPhZCQevauaJs-IkAHG8iQ.png)

After sending this request we get 50 points and we need a total of 150 points, we get points after every 24 hours.

Butt, what if i capture this request, send it to repeater, create a group and send them all together?

Classic race condition.

![](https://cdn-images-1.medium.com/max/800/1*5fUTgHYhBe-gwUdAann8ww.png)![](https://cdn-images-1.medium.com/max/800/1*YkL0a-E_phW4UM9Kawo-dQ.png)![](https://cdn-images-1.medium.com/max/800/1*Omb9i7LM0zLxuU0zQaSpDw.png)![](https://cdn-images-1.medium.com/max/800/1*HcoefsojXs8PqVr0eS2VTw.png)

We got a response, now lets check the browser if we the race condition was successful.

![](https://cdn-images-1.medium.com/max/800/1*v5ntV7MhyIABaxl6ovCOxQ.png)

Andd yeah, we got 1,050 points now we can open the vault.

![](https://cdn-images-1.medium.com/max/800/1*EEbvJz0rQR4whA9hn6IVZQ.png)

Got the flag.

Thanks for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [August 4, 2026](https://medium.com/p/503360ee713f).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-towel-on-the-sunbed-503360ee713f)

Exported from [Medium](https://medium.com) on August 8, 2026.

# TRYHACKME ROOM 404

This room is a part of THM’s HackerHolidays 2026.

---

### TRYHACKME ROOM 404

![](https://cdn-images-1.medium.com/max/800/1*C2VIQ1zQi250HfpCuoWvEg.png)

This room is a part of THM’s HackerHolidays 2026.

He booked the quiet room. It’s not on the floor plan, not in the brochure, not on any door. But port 8080 is wide open, and the rooms it never lists are the ones worth finding.

The room description tells us that port 8080 is open and the check list talks something about the accidentally available public source code.

![](https://cdn-images-1.medium.com/max/800/1*EG4stf7mjW2ZtHZVXuc77Q.png)

Lets try basic directory busting :-

![](https://cdn-images-1.medium.com/max/800/1*GpGUffKejN0tKzqaqhfGYw.png)

aaha, public .git directory.

![](https://cdn-images-1.medium.com/max/800/1*FePhzAJeOmbL1gVp06iv7A.png)

Lets try reading the files,

![](https://cdn-images-1.medium.com/max/800/1*w64zrWyXsRfvJ9Cg2xfj2w.png)

Well, it gives a 404 not found (reference to the room name itself)

butttt if we look correctly, instead of opening the /branches in the .git directory it goes back to the web root, lets try adding .git in the url ourselves.

![](https://cdn-images-1.medium.com/max/800/1*QpAuDIC4MHgX_c90CfhZqw.png)

Now, we can access each directory

Instead of going through commit history and other stuff manually one by one I decided to use gittools.

[**GitHub - internetwache/GitTools: A repository with 3 tools for pwn'ing websites with .git…**  
*A repository with 3 tools for pwn'ing websites with .git repositories available - internetwache/GitTools*github.com](https://github.com/internetwache/GitTools.git "https://github.com/internetwache/GitTools.git")

Lets use the dumper first :-

![](https://cdn-images-1.medium.com/max/800/1*uk2GogIALIDX_qCNrJMdBg.png)![](https://cdn-images-1.medium.com/max/800/1*mCuLasxfkmdafz6EF-BRAg.png)

Now we can use the extractor to extract commits from the dump.

![](https://cdn-images-1.medium.com/max/800/1*rPPsMS4xEnQhMsLhWkExJg.png)

andd, we find the app.js

Lets read it.

![](https://cdn-images-1.medium.com/max/800/1*0EZTlL761RlnF-N4ADW90A.png)

andd i got the flag!

Thanks for reading.

By [Lightningfst](https://medium.com/@lightningfst8) on [July 29, 2026](https://medium.com/p/48ab281f5ac0).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-room-404-48ab281f5ac0)

Exported from [Medium](https://medium.com) on August 8, 2026.

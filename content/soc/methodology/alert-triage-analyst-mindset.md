+++
title = "Alert Triage and the SOC Analyst Mindset"
date = "2026-06-28T00:00:00-05:00"
tags = ["soc", "triage", "incident-response", "windows-events", "sysmon", "edr", "methodology"]
description = "What I've learned about how analysts think through alert queues, ambiguous scenarios, and investigation decisions. Built from homelab work and deliberate study, not production experience."
draft = false
+++

I haven't worked a production SOC shift, so I want to be upfront about that before anyone reads this as war stories. Everything below came out of homelab work, CTF investigations, running my own simulations, and reading how real analysts talk through their process. It's not experience, it's the model I've built from studying people who have the experience.

I'm writing it down mostly because I've noticed I don't actually know something until I try to explain it in full sentences. Half of this started as a note to myself that got long enough to be a post.

---

## How I Think About a Queue Full of Alerts

Everywhere I look, the advice is "start with the highest severity." I've come around to thinking that's kind of wrong, or at least incomplete. Severity is a starting point, it's not the queue order.

The way I understand it: you triage the queue before you investigate anything. You're looking for patterns across all the open alerts before you pull any single one.

What I'd look for first, in order:

1. **Lateral movement:** if any alert suggests an attacker has moved between systems, that goes to the top immediately. The incident is actively spreading. Every other alert can wait.
2. **Critical assets:** a domain controller, a payment system, an executive's endpoint. The same behavior on a developer's laptop and a domain controller are not the same alert, regardless of how the SIEM scored them.
3. **Correlated clusters:** three alerts from the same host inside fifteen minutes, or the same source IP appearing across multiple unrelated alerts. Isolated noise is low priority. A pattern pointing at one target is not.
4. **Active vs. historical:** an alert that fired at 3 AM and is sitting untouched is historical. An alert that fired two minutes ago may still be live.

The thing I keep reminding myself: a critical-severity alert on a sandboxed test machine is probably less urgent than a medium-severity alert showing a new scheduled task on a domain controller. Severity tells you what the rule thinks. Context tells you what it means.

---

## Walking Through a PowerShell Alert

When I've worked through this type of alert in homelab simulations, including the Kerberoasting and StuxBot investigations I've documented here, the logic I've developed is to build a chain, not answer a single question.

An alert fires for a PowerShell process with a download cradle in the command line. Here's the sequence I'd work through:

**What spawned PowerShell?** Sysmon Event ID 1 captures this in the `ParentImage` field. If the parent is a Word document, an Excel file, or an Outlook process, that's a likely phishing execution chain and the urgency changes. If the parent is a scheduled task that's been running for months, the context changes completely.

**What was the full command line?** Event ID 4688 captures process creation with command line arguments if auditing is configured, but Sysmon Event ID 1 is more reliable because it doesn't depend on that policy being set. Obfuscated arguments like base64 strings, concatenated characters, and `[System.Text.Encoding]::Unicode.GetString` patterns indicate someone trying to avoid detection. That's worth noting separately from what the command actually does.

**Did the network connection fire?** Sysmon Event ID 3 records network connections per process. I'd look for an outbound connection from that PowerShell process within the same time window as the alert. If it connected out, I'd check firewall or proxy logs to see if the destination responded.

**What landed on disk?** Sysmon Event ID 11 (FileCreate) in the window after execution. If something was written to a temp directory or an AppData folder, I'd hash it immediately and check it against threat intel. Event ID 15 covers alternate data stream creation, which is used for zone identifier marking on downloaded files.

**What ran next?** Back to Sysmon Event ID 1, looking for child processes spawned by PowerShell. If PowerShell spawned something else, especially from a temp path or with encoded arguments, that's the payload executing.

The chain I follow looks like this:

```
Alert → PowerShell process (Sysmon 1) → Parent process (what spawned it)
     → Command line (4688 / Sysmon 1) → Network connection (Sysmon 3)
     → Files written (Sysmon 11/15) → Child processes (Sysmon 1)
     → Registry changes (Sysmon 12/13) → Persistence check
```

If I reach the end of that chain and everything is explainable (known parent, internal endpoint, no files dropped, no children), that's enough to close with documentation. If anything in the chain is unexpected, I have a true positive and a containment decision to make.

---

## The Phishing Scenario

A user calls and says they clicked a link. What do I do?

My understanding of the correct order here: contain first, investigate second. If the EDR supports network isolation, isolate the endpoint before you have the full picture. An attacker who already has execution on the host can move laterally while you're reading logs.

After isolation, I'd work backwards:

- **Identify the email:** sender, sending IP, the URL or attachment, and who else received it. The same email in fifty mailboxes is a very different response than a targeted single delivery.
- **Check what the URL actually served:** was it a credential harvester, a drive-by download, or a redirect chain? URLScan.io or a detonation sandbox shows the behavior without re-clicking it. If it was a credential harvesting page, I'd reset the user's credentials before I even confirmed they typed anything in. Resetting a password costs a few minutes of annoyance. Waiting to find out if credentials leaked costs a lot more if the answer is yes.
- **Check the endpoint:** what process spawned from the browser after the click? Sysmon Event ID 1. Any scripting engine, document viewer, or unexpected binary that launched from the browser in the window after the click gets treated as execution until I can prove otherwise.
- **Check authentication logs:** did the account log in from somewhere new after the click? Event ID 4624 (successful logon) with an unexpected source IP, or 4648 (logon with explicit credentials) from an unfamiliar process, means credentials were used.
- **Scope it:** did the same domain or sending IP appear in other mailboxes? Did the file hash show up on other endpoints? I'd treat it as a campaign until I can prove it was isolated.

The thing I've internalized about this scenario: waiting to confirm credential submission before resetting is the wrong instinct. By the time you confirm it, it may already matter.

---

## The Ambiguous Case

An alert fires for PowerShell execution on a finance workstation at 11 PM. It could be an admin running a script. It could be something else.

This is the hardest category to get right, and I think it's where a lot of people either over-escalate or close things too fast to keep the queue manageable. Both are wrong.

The way I've come to think about it: ask what this would look like if it were malicious, and then check whether the evidence supports that.

- If PowerShell was spawned by Task Scheduler and that scheduled task has fired at 11 PM every night for six months, that's a different story than PowerShell spawned interactively at an odd hour by a user who shouldn't be working.
- If the account is a named user on a finance workstation, is that user in a role where running PowerShell scripts is expected? Finance workstations running scripts outside business hours with no established pattern is a legitimate question.
- If the command calls an internal endpoint with a known path, that's different from encoded arguments making outbound connections.

If I work through the parent process, the account, the command, and the network activity and everything is explainable, I'd close it with documentation of what I checked and why I assessed it benign. If the same behavior fires again, that record saves the next analyst from starting from scratch.

If anything in that chain is unexpected, the investigation escalates.

What I've taken from studying this: you're not trying to prove something is malicious or prove it's clean. You're trying to build enough documented context to make a defensible call either way. The documentation matters even if you get it wrong, because it shows you were operating a process, not guessing.

---

## Event IDs I've Built Into My Reference

From building detections in Wazuh and running Windows log analysis in Splunk, these are the ones I know without looking them up:

| ID | Event |
|---|---|
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4648 | Logon with explicit credentials |
| 4768 | Kerberos TGT request |
| 4769 | Kerberos service ticket request (RC4 encryption type 0x17 = Kerberoasting indicator) |
| 4776 | NTLM credential validation |
| 4720 | User account created |
| 4732 | User added to local privileged group |
| 4688 | Process creation |
| 4698 | Scheduled task created |
| 1102 | Security audit log cleared |

Event ID 1102 is worth treating as its own escalation trigger. Log clearing after any suspicious activity isn't the end of the investigation. It's an indication that someone is covering tracks, which makes everything before it more significant.

Sysmon fills the gaps native Windows logging leaves open. Event ID 1 (process creation with full command line), Event ID 3 (network connection attributed to a specific process), Event ID 10 (LSASS access), and Event ID 22 (DNS query) have come up in nearly every investigation I've run in the homelab.

---

## What I'm Still Building

I could've written this like a how-to guide, like I've done all of it a hundred times on the job. I didn't want to fake that. Everything above is my best current model, pieced together from a homelab, other people's writeups, and a lot of deliberately breaking things to see what the logs look like afterward.

A real queue at 2am with forty open tickets is going to feel nothing like this. I know that. But the reasoning underneath it should hold up either way, and that's really the whole reason I built the lab in the first place.

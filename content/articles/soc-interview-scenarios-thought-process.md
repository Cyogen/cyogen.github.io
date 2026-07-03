+++
title = "SOC Interview Scenarios: What They're Actually Testing"
date = "2026-06-28T00:00:00-05:00"
tags = ["soc", "triage", "incident-response", "interview", "methodology", "career"]
description = "Common SOC analyst interview scenarios broken down by what they're actually testing: the fundamentals, the thought process, and the steps that matter."
draft = false
+++

A lot of SOC interviews lean hard on scenario questions. They describe an alert or an incident and just ask what you'd do. Looks like a knowledge check on the surface, but I don't think that's really what's happening. The scenario's just the vehicle, what they're actually watching for is how you think, whether you've got a repeatable process, whether you chain evidence together or treat everything as an isolated event.

I've been running through a bunch of these as prep, and writing down what I think each one is really probing for underneath the surface question.

---

## "You have 50 alerts in the queue. How do you prioritize?"

**What it's testing:** Whether you triage before you investigate, and whether you understand that severity scores are context-dependent.

The trap answer is "start with the highest severity." It sounds reasonable but it misses the point. A critical-severity alert on a sandboxed dev machine is lower priority than a medium alert on a domain controller.

The question is really asking: what mental model do you use to decide where to put your attention? A structured answer works through the queue at a meta level before pulling any individual alert. You're looking for:

- **Lateral movement:** any sign an attacker is spreading between systems. This is already an active incident. Everything else waits.
- **Asset criticality:** what is the affected system? The same behavior on a workstation and on an authentication server are different problems.
- **Correlation:** multiple alerts from the same source, or the same host appearing across several unrelated rules, is a pattern. Isolated alerts are lower priority than clusters.
- **Recency:** is this still live or did it fire at 3 AM and sit untouched? Active events take priority over historical ones.

The underlying principle: scope first, severity second. You need to know the shape of the queue before you start pulling threads.

---

## "Walk me through investigating a PowerShell alert."

**What it's testing:** Whether you investigate in a chain, not in isolation. Whether you know which data sources answer which questions. Whether you understand what "context" actually means operationally.

The steps are less important than the reasoning behind them. The question is not "what do you look at." It's "why do you look at it in that order?"

The chain I've worked out:

1. **What spawned it?** The parent process tells you intent more than almost anything else. PowerShell spawned by a scheduled task that's run every night for six months is a different story than PowerShell spawned by WINWORD.EXE. Sysmon Event ID 1 captures the parent/child relationship.

2. **What did it do?** The full command line. Obfuscation: base64 strings, concatenation tricks, encoded arguments. It indicates someone trying not to be seen. That's separate from what the command actually does, and both matter.

3. **Did it reach the network?** Sysmon Event ID 3, attributed to the PowerShell process. If an outbound connection fired, the attempt reached the network. Firewall and proxy logs tell you whether it succeeded.

4. **What landed on disk?** Sysmon Event ID 11 in the same time window. Anything written to temp directories or AppData is worth hashing and checking against threat intel.

5. **What ran next?** Child processes spawned by PowerShell after execution. If it spawned something else, especially with encoded arguments or from an unexpected path, that's the payload executing.

The important thing the question is probing: you should never be answering a single question about an alert. You're building a timeline. Each answer points to the next question. If you stop after one data point, you either miss a true positive or close one prematurely.

---

## "A user called and said they clicked a phishing link. What do you do?"

**What it's testing:** Whether you understand that containment and investigation run in a specific order. Whether you know how to scope a campaign, not just one endpoint.

The sequence matters more than any individual step. Specifically: contain first, then investigate. If the endpoint is isolated before you have the full picture, the attacker can't pivot to another system while you're reading logs. Investigation takes time. Lateral movement doesn't wait.

After containment:

- **Scope the email:** who else got it? One target is different from a hundred. Pull the sending IP, domain, URL, and any attachments.
- **Check what the URL served:** credential harvester, drive-by download, or redirect chain? That determines the response. A credential harvesting page means a password reset happens immediately, before you even confirm the user typed anything in. A reset is a minor inconvenience. Waiting to find out is not.
- **Check the endpoint:** what spawned from the browser after the click? What files were written? What network connections fired?
- **Check auth logs:** did the account log in from an unexpected location after the click? Did an unfamiliar process use the user's credentials?
- **Treat it as a campaign:** search the same IOCs across all endpoints and mailboxes before you close the scope.

The fundamental the question is looking for: you contain before you understand. Waiting for certainty before acting is the wrong instinct in an active incident.

---

## "How do you handle a false positive that keeps firing?"

**What it's testing:** Whether you understand the tradeoff between tuning and blind spots. Whether you document decisions rather than just making them.

The scenario is not really about false positives. It's about whether you understand that every suppression you add to a rule is a potential gap in coverage, and whether you treat that with appropriate seriousness.

The process:

1. **Identify the legitimate behavior** causing the false positive. What is the rule detecting? What is the benign activity that matches it?
2. **Tune around that behavior specifically,** not broadly. Suppress by process lineage, by source account, by time window, or by asset group, whichever is most precise. The goal is to exclude the known-good case without expanding the exclusion further than necessary.
3. **Document the decision:** what you suppressed, why, and what verification you did. This matters because exclusions age. Six months from now, someone will look at that suppression and need to know if it's still valid.
4. **Monitor after tuning:** verify that the false positive case is excluded and that the rule still fires on activity that should trigger it.

The underlying principle: you are not trying to eliminate the alert. You are trying to eliminate the noise without eliminating the signal. Those are not the same operation.

---

## "What would you do if you weren't sure whether something was malicious?"

**What it's testing:** Whether you can operate under uncertainty without either over-escalating or dismissing things to keep the queue clean.

The temptation in both directions is real. Escalating everything uncertain avoids responsibility but generates noise and erodes trust in the team's judgment. Closing things as false positives because you can't immediately prove malice is how true positives get missed.

The framework I've worked out:

- **State what you know.** What fired, what asset is involved, what the initial indicator is.
- **State what you don't know.** What additional context would change the assessment.
- **State what you'd look at next** to answer what you don't know.
- **State what you'd do right now** while you keep investigating: monitor, contain, or hold.

The question they're probing with this scenario: can you make a documented, defensible call without certainty? The answer "I don't know" is never acceptable on its own. The answer "I don't know yet, and here is what I would do to find out" is the right frame.

And crucially: document the reasoning even if you close it as benign. If it fires again, the record tells the next analyst you already looked at this. If it turns out you were wrong, the documentation shows you were following a process, not guessing.

---

## The Thread Running Through All of These

Strip away the specifics and it's the same question every time, dressed up differently: do you have a repeatable, documented way of moving through ambiguity, or are you improvising each time.

Anyone can memorize the steps. What's harder to fake is the instinct underneath them, building a chain instead of stopping at one data point, containing before you fully understand, writing down why you made a call instead of just making it.

If there's one line worth keeping in your back pocket for these, it's that "I don't know" on its own is a dead end. "I don't know yet, here's how I'd find out" is a completely different answer.

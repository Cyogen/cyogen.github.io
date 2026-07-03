+++
title = "TheHive"
date = "2026-06-29T00:00:00-05:00"
tags = ["thehive", "case-management", "incident-response", "soc", "homelab"]
description = "Open-source case management for SOC workflows. Running in Docker, integrated with Wazuh for automatic case creation on significant alerts."
draft = false
+++

Open-source incident response and case management platform. Wazuh handles the log ingestion and alerting side, TheHive picks up from there, opening the case, tracking the investigation, logging observables, writing up what actually happened.

Before I set this up, my "case management" was a .txt template sitting in a folder on a second drive. It worked in the sense that the info existed somewhere, but it had zero connection to the actual alerts and nothing about it was searchable. TheHive fixed all of that.

It runs in Docker and talks to Wazuh through a custom integration script I wrote. Anything level 5 or above opens a case automatically, already tagged with the source agent, rule group, and MITRE technique. I don't touch anything manually unless it's actually worth my time.

## Utility

**Observable correlation.** Attach indicators to cases: IPs, hashes, domains, accounts. If the same observable appears across multiple cases, TheHive surfaces the relationship automatically.

**Case templates.** Reusable task sets per alert type. A Kerberoasting case opens with tasks already defined for log review, DC log check, source account audit, and verdict. Consistent process every time without rebuilding it from scratch.

**MITRE tagging.** Cases carry the ATT&CK tags from the Wazuh rule. Over time the case history maps to actual techniques investigated, not just rules written.

## Where It's At

Fully up and running. The Wazuh integration is live, every significant alert opens its own case without me lifting a finger, and I work the queue the same way I'd expect to in an actual job.

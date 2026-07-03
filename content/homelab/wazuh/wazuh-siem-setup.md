+++
title = "Building a SIEM from Scratch: Wazuh on Dedicated Hardware"
date = "2026-06-25T10:00:00-05:00"
tags = ["wazuh", "siem", "homelab", "debian", "active-directory", "soc"]
description = "Installing Wazuh all-in-one on a Debian server, registering agents across Arch Linux and Windows Server 2022, and documenting every failure along the way."
draft = false
+++

Every SOC needs a SIEM somewhere at the center of it, and for this lab I went with Wazuh, it's open source and handles log collection, threat detection, and alerting across whatever's on the network. This is the full install on a dedicated Debian box, hooking up agents on Arch Linux and a Windows Server VM, and every single thing that went wrong along the way, because plenty did.

## Hardware

The SIEM server is a repurposed machine running Debian stable: 16GB RAM,
500GB NVMe, living on the LAN at `10.0.42.114` with no GUI. Just SSH and a
display attached for local access when needed.

## The Problem with 4K TTY

First obstacle before installing anything: the TTY font on a 4K display is
microscopic. Completely unreadable without glasses you don't own.

```bash
sudo apt install console-setup fonts-terminus
setfont ter-132b
```

To make the large font persist across reboots:

```bash
sudo dpkg-reconfigure console-setup
```

Select: UTF-8 → Latin → Terminus → largest available size.

## Installing Wazuh

Wazuh provides a single all-in-one install script that deploys the manager,
OpenSearch indexer, and dashboard together. Download and run it:

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

This takes several minutes. When it finishes, credentials are saved to
`wazuh-install-files.tar`. Extract them:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

The dashboard should be reachable at `https://10.0.42.114`.

![Fresh Wazuh dashboard after install](/images/FreshWazuhLogin.png)

## Issue: Dashboard Stuck: ECONNREFUSED

After install the dashboard stayed on "not ready yet" for 10+ minutes.
The logs explained why:

```
[ConnectionError]: connect ECONNREFUSED ::1:9200
```

The dashboard was trying to reach the OpenSearch indexer on IPv6 loopback
(`::1`). The indexer wasn't listening there: it was bound to the LAN IP.

Fix in `/etc/wazuh-dashboard/opensearch_dashboards.yml`:

```yaml
# Wrong: resolves to ::1 on this system
opensearch.hosts: ["https://localhost:9200"]

# Also wrong: indexer isn't on loopback in an all-in-one install
opensearch.hosts: ["https://127.0.0.1:9200"]

# Correct: indexer binds to the LAN interface
opensearch.hosts: ["https://10.0.42.114:9200"]
```

Verify where the indexer is actually listening before assuming:

```bash
ss -tlnp | grep 9200
```

```bash
sudo systemctl restart wazuh-dashboard
```

## Issue: OpenSearch Causing Constant Fan Spin

With the dashboard up, the server fans were spinning continuously under no
meaningful load. The culprit was the default JVM heap size for the indexer -
1024MB. A heap that small causes constant garbage collection cycles, which
reads as sustained CPU usage.

Fix in `/etc/wazuh-indexer/jvm.options`:

```
-Xms2g
-Xmx2g
```

```bash
sudo systemctl restart wazuh-indexer
```

Fans dropped back to idle within a minute. 2GB is a good balance for a
small lab: enough headroom to avoid GC thrash, not so much it starves
everything else on a 16GB machine.

## Registering the First Agent: Arch Linux Workstation

Wazuh has no official Arch Linux package. The AUR covers it:

```bash
yay -S wazuh-agent
```

Point the agent at the manager in `/var/ossec/etc/ossec.conf`:

```xml
<server>
  <address>10.0.42.114</address>
  <port>1514</port>
  <protocol>tcp</protocol>
</server>
```

Register and start:

```bash
sudo /var/ossec/bin/agent-auth -m 10.0.42.114
sudo systemctl enable --now wazuh-agent
```

## Issue: Agent Version Newer Than Manager

The AUR package pulled Wazuh 4.14.5. The fresh install deployed 4.12.0.
Wazuh requires agent version ≤ manager version, so registration failed:

```
ERROR: Agent version must be lower or equal to manager version
```

The fix is to upgrade the manager immediately after every fresh install -
the installer packages lag behind the AUR:

```bash
sudo apt-get update
sudo apt-get install --only-upgrade wazuh-manager wazuh-indexer wazuh-dashboard
sudo systemctl restart wazuh-manager wazuh-indexer wazuh-dashboard
```

With versions aligned, the Arch Linux workstation agent connected and appeared in the
dashboard.

![Arch Linux agent connected](/images/wazuh-agent_7900xconnected.png)

## Critical: Never Install wazuh-agent on the Manager Machine

While setting up monitoring for the SIEM server itself, there was a temptation to
install `wazuh-agent` on the same machine as the manager. Don't.

The `wazuh-manager` and `wazuh-agent` packages conflict: installing one
removes the other. The manager already monitors itself via a built-in local
agent (ID 000). No separate agent package is needed or wanted on the manager.

Installing `wazuh-agent` on the SIEM server wiped the manager package and reset the
API security database, invalidating all credentials. Recovery required a full
reinstall with `--overwrite`:

```bash
sudo bash wazuh-install.sh -a --overwrite
```

Then upgrade again immediately after. Lesson logged.

## Registering the Second Agent: Windows Server 2022 (DC01)

DC01 is a Windows Server 2022 VM running Active Directory on the lab network
(`192.168.122.10`). Install the Wazuh agent via PowerShell as Administrator:

```powershell
Invoke-WebRequest `
  -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi `
  -OutFile $env:tmp\wazuh-agent

msiexec.exe /i $env:tmp\wazuh-agent /q `
  WAZUH_MANAGER='10.0.42.114' `
  WAZUH_AGENT_NAME='DC01'

NET START WazuhSvc
Set-Service -Name WazuhSvc -StartupType Automatic
```

Both agents confirmed active in the dashboard.

![Both agents: Arch and DC01: connected](/images/wazuh-agents-arch_DC.png)

## Where Things Stand

Three endpoints reporting into Wazuh:

| Agent | OS | Role |
|---|---|---|
| Arch Linux Workstation | Arch Linux | Daily driver / lab host |
| DC01 | Windows Server 2022 | Active Directory domain controller |
| SIEM Server | Debian (local agent 000) | Wazuh manager itself |

So the SIEM's collecting logs, running rules, and throwing alerts now. What it doesn't have yet is anywhere to actually track what needs action, that's TheHive's job, and I get into setting that up in the next post.

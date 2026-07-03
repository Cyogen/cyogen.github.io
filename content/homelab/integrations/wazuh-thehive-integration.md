+++
title = "Connecting Wazuh to TheHive: Automatic Case Creation from SIEM Alerts"
date = "2026-06-25T18:00:00-05:00"
tags = ["wazuh", "thehive", "homelab", "soc", "integration", "python", "case-management"]
description = "Wiring Wazuh's custom integration framework to TheHive's REST API so every alert above a severity threshold automatically opens a tracked case."
draft = false
+++

Wazuh's generating alerts, TheHive's sitting there ready to manage cases, and
the only thing missing is actually wiring the two together. This post is that
bridge, a Python script that lives on the Wazuh manager and fires alerts into
TheHive as cases the instant they trip.

## How Wazuh Integrations Work

Wazuh has a built-in integration framework. Any executable placed in
`/var/ossec/integrations/` with a `custom-` prefix gets called automatically
when an alert matches the configured filter. Wazuh passes three arguments:

1. Path to a JSON file containing the full alert
2. API key (from `ossec.conf`)
3. Hook URL (from `ossec.conf`)

The script handles the rest. No daemon, no service: just a script that runs
per alert.

## TheHive API Key

In TheHive, go to **Admin → Users**, select the admin account, and generate
an API key. Copy it: this goes into the Wazuh config.

## The Integration Script

On the **Wazuh manager (`10.0.42.114`)**:

```bash
sudo nano /var/ossec/integrations/custom-thehive
```

```python
#!/usr/bin/env python3
import sys
import json
import requests
from datetime import datetime

def main():
    alert_file  = sys.argv[1]
    api_key     = sys.argv[2]
    thehive_url = sys.argv[3]

    with open(alert_file) as f:
        alert = json.load(f)

    rule  = alert.get("rule", {})
    level = rule.get("level", 0)
    agent = alert.get("agent", {})

    if level <= 2:
        return

    severity = 1
    if level >= 10:
        severity = 3
    elif level >= 7:
        severity = 2

    title = f"[Wazuh] {rule.get('description', 'Alert')}: {agent.get('name', 'unknown')}"
    description = (
        f"**Rule:** {rule.get('id')}: {rule.get('description')}\n"
        f"**Level:** {level}\n"
        f"**Agent:** {agent.get('name')} ({agent.get('ip', 'N/A')})\n"
        f"**Time:** {alert.get('timestamp', datetime.utcnow().isoformat())}\n\n"
        f"```json\n{json.dumps(alert, indent=2)}\n```"
    )

    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }

    case = {
        "title": title,
        "description": description,
        "severity": severity,
        "tlp": 2,
        "tags": ["wazuh", f"rule:{rule.get('id')}", f"level:{level}"],
        "flag": False
    }

    requests.post(
        f"{thehive_url}/api/case",
        headers=headers,
        json=case,
        timeout=10
    )

if __name__ == "__main__":
    main()
```

Set permissions: Wazuh runs integrations as the `wazuh` user:

```bash
sudo chmod 755 /var/ossec/integrations/custom-thehive
sudo chown root:wazuh /var/ossec/integrations/custom-thehive
```

## Severity Mapping

The script maps Wazuh's 0–15 rule levels to TheHive's 1–3 severity scale:

| Wazuh Level | TheHive Severity |
|---|---|
| 3–6 | 1: Low |
| 7–9 | 2: Medium |
| 10–15 | 3: High |

Levels 0–2 are skipped entirely: those are informational Wazuh events, not
actionable alerts.

## Configuring Wazuh to Call the Script

Add the integration block to `/var/ossec/etc/ossec.conf` just before the
closing `</ossec_config>` tag:

```xml
<integration>
  <name>custom-thehive</name>
  <hook_url>http://10.0.42.139:9000</hook_url>
  <api_key>YOUR_THEHIVE_API_KEY</api_key>
  <level>5</level>
  <alert_format>json</alert_format>
</integration>
```

`<level>5</level>` means only alerts at severity 5 or above trigger case
creation. This keeps minor events out of the case queue while ensuring
anything meaningful gets tracked.

Restart the manager to apply:

```bash
sudo systemctl restart wazuh-manager
```

## What Happened When I Restarted It

Six cases showed up in TheHive right after the restart, before I'd even done anything else. Turned out Wazuh had queued alerts from agents that were already generating events, and it just backfilled the whole batch on its own. Didn't have to trigger anything manually.

![TheHive case queue populated from Wazuh alerts](/images/TheHive_caseIntegration.png)

So the pipeline looks like this now:

```
Agent (Arch Linux Workstation / DC01 / SIEM Server)
  → Wazuh Manager
    → custom-thehive script
      → TheHive REST API
        → Case queue
```

Anything level 5 or above lands in TheHive with the full alert JSON attached, severity already mapped, agent context included, tagged by rule ID and level so I can filter later. From there I can assign it, work it, close it out with an actual timeline instead of just a Slack message to myself.

Feels like a real SOC workflow at this point, just running on hardware sitting in my apartment. Next thing on the list is generating actual detections to test it against, probably Atomic Red Team against the AD lab.

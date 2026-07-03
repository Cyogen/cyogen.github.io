+++
title = "Delete or Suppress? Handling False Positives in a SOC"
date = "2026-06-26T00:00:00-05:00"
tags = ["soc", "wazuh", "splunk", "false-positive", "incident-response", "log-management"]
description = "After closing a false positive alert, you're left with a decision: delete the event from the SIEM or suppress it at the rule layer. The answer matters more than you think."
draft = false
+++

After [investigating the Wazuh rootcheck alert on `/usr/bin/chsh`](/soc/wazuh-rootcheck-false-positive-chsh/), I had a confirmed false positive just sitting there in my queue. Binary's clean, rule's tuned, it's never going to fire again.

Which left me with one actual question: what do I do with the alert that's already there?

My gut said delete it, obviously, it's noise, it's wrong, why would I keep it around. Turns out that instinct is usually wrong in a real SOC.

## Two Options, Different Implications

### Option 1: Delete the Event

In **Wazuh**, historical alerts live in OpenSearch indices. You can remove
them with a `_delete_by_query` call:

```bash
curl -k -u admin:PASSWORD -X POST \
  "https://WAZUH_IP:9200/wazuh-alerts-*/_delete_by_query" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "must": [
          { "term": { "rule.id": "510" } },
          { "match": { "rule.description": "chsh" } }
        ]
      }
    }
  }'
```

In **Splunk**, you delete events using the `| delete` search command
(requires the `delete` role):

```
index=main rule_id=510 "chsh" | delete
```

Both work. Both permanently remove the event from the SIEM.

### Option 2: Suppress the Alert

Suppression leaves the raw log intact but stops it from generating noise.

In **Wazuh**, this is a custom rule that overrides the original at `level="0"`:

```xml
<group name="rootcheck,">
  <rule id="100001" level="0">
    <if_sid>510</if_sid>
    <match>chsh</match>
    <description>FP: /usr/bin/chsh verified clean by dpkg on Debian</description>
  </rule>
</group>
```

The alert no longer fires. The original log is untouched in the indexer.

In **Splunk Enterprise Security**, this is a Notable Event suppression. You
search for the false positive, flag it, define the suppression criteria, and
it stops appearing in the analyst queue: while the raw event stays in the
index forever.

## Why the Difference Matters

This isn't a technical preference. It's an operational and legal one.

**Logs are evidence.** Even a false positive alert represents something that
actually happened on a system at a specific time. In a real environment,
those logs may be subject to:

- **Retention policies**: compliance frameworks like PCI-DSS, SOC 2, and
  HIPAA mandate minimum log retention periods. Deleting events can put the
  organization out of compliance even if the events themselves were harmless.
- **Forensic timelines**: during an incident investigation, analysts build
  timelines of everything that happened on a system. A gap where logs were
  deleted creates ambiguity. Was it a false positive that got cleaned up, or
  did someone remove evidence?
- **Audit trails**: in regulated industries, auditors want to see an
  unbroken log record. Selective deletion raises questions.

**Suppression keeps the evidence. It just stops it from generating work.**

The chsh alert will always show in the raw OpenSearch data if someone goes
looking. What it won't do is wake up an analyst at 2 AM or inflate the
false positive rate metrics. That's the right outcome.

## When Deletion Is Appropriate

Deletion isn't always wrong. There are legitimate cases:

- **Test data contamination**: you ran agent tests or scans that flooded
  the SIEM with thousands of irrelevant events before the environment was
  production-ready. Cleaning those up before go-live is reasonable.
- **PII exposure**: a misconfigured log source accidentally ingested data
  it shouldn't have (passwords, SSNs, card numbers). Deletion may be
  required by policy to contain the exposure.
- **Storage constraints**: in a resource-limited homelab, purging old
  low-value logs to free up indexer space is a practical tradeoff.

In all three cases, deletion is a deliberate, documented decision: not a
reflex to make the dashboard look clean.

## What I Actually Did

Went with suppression on the chsh one, not deletion.

Rule's tuned, and the event just sits there in OpenSearch untouched. If I ever need to go back and check what rootcheck was doing on the box in those first few hours after deployment, it's still there. And if some future version of me, or anyone else, wonders why there's a suppression rule for chsh sitting in local_rules.xml, the original alert answers that on its own.

Queue stays clean, evidence stays intact. That's the outcome I want out of a SOC, doesn't matter if it's a homelab or a real one.

## The Takeaway

Closing out a false positive is really a decision about log integrity whether you think of it that way or not. My default now is:

1. **Investigate** it enough to actually confirm it's a false positive
2. **Tune** it at the rule layer so it stops recurring
3. **Document** why I suppressed it and what I checked
4. **Retain** the original event in the index no matter what

Deletion only makes sense when there's a real documented reason behind it, not just "it's cluttering my dashboard."

**Alert status:** Suppressed, false positive
**Data retained:** Yes, original event preserved in the OpenSearch index

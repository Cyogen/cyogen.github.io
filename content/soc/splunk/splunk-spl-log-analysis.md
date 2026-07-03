+++
title = "SPL Log Analysis: Detection Queries in Splunk"
date = "2026-06-05T00:00:00-05:00"
tags = ["splunk", "spl", "detection", "log-analysis", "windows-events", "kerberos", "homelab"]
description = "Hands-on SPL query development for Windows security log analysis in Splunk, including Kerberos ticket analysis, logon tracking, and brute force detection."
draft = false
+++

Notes from a lab exercise where I actually sat down and wrote SPL against Windows 11 event logs forwarded through a pfSense network on Splunk Enterprise. Wanted to build real detection queries myself instead of just running whatever came pre-packaged.

## Environment

- Splunk Enterprise (trial)
- Windows 11 Professional (log source)
- pfSense (network)
- Windows Security event log forwarding to Splunk

## Splunk Orientation

A few things worth knowing before writing queries:

**Data Sources:** Settings > Data Inputs shows what is being collected and from where.

**Search Modes:** Fast mode strips field extraction for speed. Verbose mode shows every extracted field and the raw event. For learning what fields exist on an event type, use Verbose.

**Fields:** "Selected Fields" always appear in the event list. "Interesting Fields" appear in at least 20% of events. When investigating a new event type, expand the fields sidebar to understand what is available before writing a stats query.

**Data Models:** Settings > Data Models gives hierarchical views of structured data. Useful for CIM-compliant data and for writing `tstats` queries later.

---

## Query 1: Kerberos Authentication Ticket Volume by Account

Find the account generating the most Kerberos ticket requests:

```spl
index=* sourcetype="WinEventLog:Security" EventCode=4768
| stats count by Account_Name
| sort -count
```

- `index=*` includes all indexes so nothing is excluded
- `EventCode=4768` is the Kerberos Authentication Service ticket request (TGT), the first step in the Kerberos flow
- `stats count by Account_Name` collapses results into a count per account
- `sort -count` orders from highest to lowest

A single account dominating this event is normal for service accounts that authenticate constantly. An account that normally sits at the bottom suddenly appearing at the top is worth investigating.

![Kerberos ticket volume query results in Splunk](/images/posts/spl-lab/kerberos-query.png)

---

## Query 2: SYSTEM Account Logons Per Computer

Count how many times each computer was accessed under the SYSTEM account using successful logon events:

```spl
index=* sourcetype="WinEventLog:Security" EventCode=4624 Account_Name=SYSTEM
| stats count by ComputerName
| sort -count
```

Event 4624 is a successful logon. Filtering to SYSTEM accounts and grouping by machine gives a baseline for normal SYSTEM activity across the environment. A machine that suddenly shows up here that did not appear before warrants a closer look.

![SYSTEM account logon count per computer](/images/posts/spl-lab/system-logon-query.png)

---

## Query 3: Brute Force Detection Within 10-Minute Windows

The hardest query: identify which account made the most logon attempts within any 10-minute span:

```spl
index=* sourcetype="WinEventLog:Security" EventCode=4624
| stats min(_time) as first_login max(_time) as last_login count as login_attempts by Account_Name
| where (last_login - first_login) <= 600
| sort -login_attempts
```

Breaking it down:

- `min(_time) as first_login` finds the earliest logon event for each account in the search window
- `max(_time) as last_login` finds the latest
- `count as login_attempts` counts total events per account
- `where (last_login - first_login) <= 600` filters to accounts whose entire observed logon activity fits within 600 seconds (10 minutes)
- `sort -login_attempts` shows highest volume first

![Brute force detection query grouped by 10-minute windows](/images/posts/spl-lab/10min-brute-query.png)

An account with 200+ logon events within a 10-minute window when first and last timestamps are only 8 minutes apart is a brute force pattern. Combined with checking 4625 (failed logons) alongside 4624, this surfaces both successful and failed spray attempts.

---

## SPL Patterns Worth Keeping

**Time-bucket grouping** (for trend analysis):

```spl
| bucket _time span=10m
| stats count by _time, Account_Name
```

**Rate-based anomaly detection** (spike detection):

```spl
| timechart span=5m count by EventCode
```

**Joining failed and successful logons** (spray detection):

```spl
index=* sourcetype="WinEventLog:Security" (EventCode=4624 OR EventCode=4625)
| eval outcome=if(EventCode==4624,"success","failure")
| stats count by Account_Name, outcome, src_ip
| sort -count
```

This query groups by account, outcome, and source IP. A pattern of many failures from one source IP with a small number of successes is a textbook spray.

---

## Splunk Alert Configuration

Detection rules become alerts by saving a search and setting:

- **Schedule:** every 5 minutes (or matching the detection window)
- **Trigger condition:** number of results greater than threshold
- **Alert action:** email, webhook, or TheHive integration

For the brute force query above, a threshold of 50 login attempts within a 10-minute window would catch most spray activity while staying above normal single-user logon noise.

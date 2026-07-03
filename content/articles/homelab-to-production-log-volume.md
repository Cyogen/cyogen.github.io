+++
title = "The Homelab to Production Gap: Log Volume and Retention Costs"
date = "2026-06-28T00:00:00-05:00"
tags = ["soc", "splunk", "elasticsearch", "wazuh", "siem", "homelab", "production", "opinion"]
description = "At homelab scale, retention is just a disk question. At production scale, log volume and storage costs are where most SOC pipelines break down. Here is what changes and why it matters."
draft = false
+++

Someone commented on a LinkedIn post I made asking how I'm handling log volume and retention costs as the homelab scales up. Good question, and it stuck with me enough that I wanted to actually sit down and answer it properly instead of a one-liner reply.

It's a good question specifically because it's not a hard one to answer technically, it's a hard one to have actually thought about. It pokes right at the blind spot a homelab creates.

When everything's running on hardware you own with no licensing bill and effectively free storage, you just don't build the instinct for what actually constrains a real SOC. You can build a pipeline that's technically correct and still have zero feel for why a company filters certain sources before they ever hit the SIEM, or why retention policies exist to control cost as much as satisfy compliance. Basically: do I understand the tradeoffs production runs on, or do I just know how to click the buttons in isolation. Different question entirely.

Worth asking yourself that while you're still building, not after. Here's where I actually stand on it.

## At Homelab Scale, None of This Bites

Four Wazuh agents on my own hardware produce a trickle of logs I barely notice. Retention here is purely a disk question, running low means buy more disk or shrink the window. No licensing bill tied to ingestion. Nobody from an infra team asking why my indices are hammering IOPS. No compliance rule forcing me to keep 36 months of anything.

The homelab is where the tools and the detection logic click into place, and that's fine, that's what it's for. The point isn't to feel the cost pressure now, it's to understand the architecture well enough to see what breaks once the scale changes.

## What Actually Changes at Scale

A busy domain controller alone can throw [300 to 500 events per second](https://content.solarwinds.com/creative/pdf/Whitepapers/estimating_log_generation_white_paper.pdf). A normal workstation is more like 1 to 5. Put 1,000 endpoints together and even conservative logging settings add up fast, roughly [1,000 EPS works out to about 8.6 GB a day](https://www.linkedin.com/posts/shahabit_siem-sizing-is-all-about-estimating-the-resources-activity-7415739284315480064-dLow). Across a mid-size org that's dozens to hundreds of gigabytes daily before anyone's even tuned anything.

And it's not a fixed target either, [volumes are climbing roughly 50% year over year](https://securityboulevard.com/2025/05/reducing-siem-costs-with-a-security-data-fabric-a-practical-guide/).

## Where Splunk Gets Expensive

Splunk's classic licensing is priced off daily ingestion. [List pricing starts around $1,800 a year per 1 GB/day](https://www.vendr.com/marketplace/splunk), scaling up from there. Ingest 100 GB a day and you're looking at [something like $150,000 a year just in licensing](https://securityboulevard.com/2025/05/reducing-siem-costs-with-a-security-data-fabric-a-practical-guide/), before you've paid for infrastructure or anyone's time to run it.

That's the actual reason ingestion discipline matters, it's not just a nice-to-have. What gets sent to Splunk, what gets filtered at the forwarder before it ever arrives, what gets archived somewhere cheaper instead, those decisions land directly on the invoice. None of that registers when you're running Splunk Enterprise on a trial license at home. It's the very first conversation in production.

## Index Lifecycle Management in Elasticsearch

Wazuh stores its data in Elasticsearch, which has a built-in answer to the retention cost problem: [Index Lifecycle Management](https://www.elastic.co/docs/manage-data/lifecycle/index-lifecycle-management) and [data tiers](https://www.elastic.co/docs/manage-data/lifecycle/data-tiers).

The architecture works like this:

- **Hot tier**: recent data, actively searched, on fast storage (SSDs). This is where everything lands first.
- **Warm tier**: data from recent weeks, queried less frequently, can move to cheaper hardware.
- **Cold tier**: infrequently accessed data, still searchable, minimal resources needed.
- **Frozen tier**: searchable snapshots in object storage. Query performance is slower since data loads on demand, but [storage cost drops to roughly 10 to 20 times cheaper than hot-tier SSD pricing](https://www.elastic.co/docs/manage-data/lifecycle/data-tiers).

ILM policies handle the transitions automatically based on index age and size, you set the rules once and Elasticsearch does the moving. With a handful of agents at home there's genuinely no reason to touch any of this. At production scale, skipping it just means everything sits on expensive hot-tier storage forever because nobody ever told it to move.

## Noise Is the Other Half of This Problem

Volume isn't just a storage cost, it's a noise problem too. A detection rule that behaves fine on four endpoints can throw thousands of alerts a day once you're at a thousand endpoints, and most of those will be false positives until someone tunes it. A SOC drowning in noise stops working pretty fast, doesn't matter how good the detection logic is underneath.

Tuning at scale is its own skill, adjusting thresholds, suppressing known-good behavior, carving out exclusions without leaving gaps, tracking false positive rates over time. The homelab is actually a decent place to build that instinct, honestly. Adding agents, breaking things on purpose, watching what fires and what doesn't, that part does transfer.

## Where I Land on This

The homelab was never going to be a production SOC and I don't think anyone reading this expected it to be. The value is in understanding, ahead of time, which architecture decisions actually matter once the scale changes, instead of learning it the hard way with a $150,000 line item.

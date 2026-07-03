+++
title = "Splunk Boss of the SOC"
date = "2026-06-29T00:00:00-05:00"
tags = ["splunk", "soc", "training", "blue-team", "investigation"]
description = "Splunk's blue team investigation dataset. Real attack data, real SPL, real analyst workflow. One of the best hands-on training environments available."
draft = false
+++

Splunk's own blue team training dataset and competition. BOTS drops you into a realistic environment with real attack scenarios and just expects you to investigate them in Splunk. No walkthrough, nobody holding your hand. Just the data, a question, and a search bar.

It's not really a CTF in the usual sense, no flags to pop, nothing to exploit. It's pure investigation, correlating logs, tracing what an attacker actually did through the data, figuring out what happened and how. Basically the SOC analyst job compressed down into something you can practice.

## Why It's Actually Good

**The data isn't sanitized.** BOTS is built from real attack campaigns run against a real environment, so the noise and artifacts and lateral movement traces look like actual production logs, not a tidy teaching example.

**You have to write the searches yourself.** No dashboard someone already built for you to click through. It's rough at first, but the SPL gets faster and the field names start feeling familiar pretty quick once you're doing it for real.

**One scenario covers a lot of ground.** A single investigation might touch phishing, execution, persistence, credential access, and exfiltration before it's over. Working through the whole chain in one sitting is what actually builds the instinct for how these attacks progress.

## Versions

There's v1, v2, and v3 so far, each built around a different campaign, and each one gets harder than the last. You can run them in a local Splunk instance or through the Splunk Attack Range. Starting at v1 and working up is the way to go, jumping straight to v3 would just be frustrating.

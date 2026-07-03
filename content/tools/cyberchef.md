+++
title = "CyberChef"
date = "2026-06-29T00:00:00-05:00"
tags = ["cyberchef", "ctf", "analysis", "obfuscation", "malware"]
description = "Browser-based data transformation tool. The first tab open on any alert involving obfuscated commands or encoded payloads."
draft = false
+++

Browser-based data transformation tool that GCHQ of all people built and open sourced. You throw encoded or mangled data at it and chain operations together until it's readable, no scripting, no install, no copy-pasting between five different tools.

I use this constantly, both in CTFs and the second an alert shows up with an obfuscated command or a weird string in it. If a PowerShell alert fires with a base64 blob sitting in the command line, this is the tab I open before anything else.

## Utility

**Recipe chaining.** Operations stack in order. A single recipe can From Base64 → Gunzip → Extract URLs, turning a one-line encoded dropper into a readable payload with the C2 address visible. Save the recipe and replay it on the next sample.

**Magic.** Paste unknown data and let Magic detect the encoding automatically. Useful when you don't know if you're looking at base64, hex, URL encoding, or something layered. It tries combinations and scores the results by entropy and readability.

**XOR brute force.** Single-byte XOR is common in shellcode and simple malware obfuscation. CyberChef's XOR Brute Force operation tries all 256 keys and renders the output, making it trivial to spot the readable result.

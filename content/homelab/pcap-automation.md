+++
title = "PCAP Automation: Building a Scapy-Based Triage Pipeline"
date = "2026-06-08T00:00:00-05:00"
tags = ["pcap", "python", "scapy", "automation", "wireshark", "tshark", "dns", "network", "homelab"]
description = "Automated PCAP analysis pipeline using Scapy and tshark. Feed it a capture file and get back structured output covering protocols, conversations, HTTP requests, DNS anomalies, and a combined HTML report."
draft = false
+++

I got tired of doing the same manual PCAP triage every single time, top protocols, top talkers, HTTP POST inspection, DNS anomaly check, file extraction, so I automated the first pass. Same checks, just in seconds instead of five minutes of clicking around.

Source: [github.com/Alchemestrum/PCAP_automation](https://github.com/Alchemestrum/PCAP_automation)

## What It Does

Two entry points:

```bash
python3 scapy_checks.py sample.pcap    # initial Scapy analysis
./pcap.sh sample.pcap                  # full pipeline, generates results directory
```

The shell script runs the full suite and writes output to `results/<pcap_name>/`. Everything is structured for manual pivot after the automated pass.

## Output Structure

```
results/
  sample_pcap/
    summary_report.txt
    conn_log.csv
    summaries/
      http_requests.tsv
      dns_long_qnames.txt
      dns_queries.tsv
      suricata_logs/
    files/
    decoded_dns/
```

## Interpreting Results

### summary_report.txt

Top protocols and top endpoints. The first place to look. If an unexpected protocol appears here, pivot immediately to conversations before going further.

### conn_log.csv

All connections sorted by time and bytes. Two patterns to look for:

- **Persistent small flows at regular intervals:** this is beaconing. A host checking in to a C&C server will show a recurring connection with small, consistent byte counts.
- **Large outbound transfers:** exfil candidates. A sudden spike in outbound bytes to an unfamiliar destination is the obvious one, but also watch for sustained slow exfil spread across many small connections.

### summaries/http_requests.tsv

All captured HTTP requests. The focus here is on POST requests: inspect content-type and body. Hex-encoded or base64-encoded POST bodies are a strong indicator of data staging or C&C communication. Any matching payloads can be saved and decoded separately.

### summaries/dns_long_qnames.txt

DNS queries with unusually long subdomains. Normal DNS does not produce 40-character random-looking subdomains. If this file has content, something is either tunneling data through DNS or using DNS for C&C communication.

### files/

Reconstructed binaries and documents from the capture. Anything here should go into a sandbox before being opened. DOCX files in this directory can contain macros.

### summaries/suricata_logs/

IDS matches if Suricata was run against the capture. Confirm each hit by pulling the matching packets in Wireshark and verifying the signature triggered correctly. Not every Suricata hit is real, but Suricata hits combined with other indicators (anomalous DNS, unusual conversation volumes) are strong.

---

## DNS Subdomain Decoder

A separate script handles DNS anomaly decoding:

```bash
python3 decode_dns_subdomains.py results/sample_pcap/summaries/dns_queries.tsv
```

It reads the tshark DNS query output, flags subdomains over a length threshold, and tries base64, base32, and hex decoding on each one. Decoded output writes to `results/<pcap>/decoded_dns/`. If anything decodes to readable content, that is either a tunneling tool or a C&C protocol using DNS as the transport.

---

## HTML Report

After the full pipeline runs:

```bash
python3 generate_report.py
```

This combines everything into a single clickable HTML file:

- Summary statistics
- Top protocols, endpoints, and conversations
- HTTP requests and responses
- DNS queries with decoded links where applicable
- TLS session info
- Reconstructed files
- Decoded DNS artifacts

The HTML output is useful for sharing findings without requiring the recipient to have any of the tools installed.

---

## Wireshark Triage Filters (Quick Reference)

These complement the automated output for manual follow-up:

```
tcp.flags.syn==1 && tcp.flags.ack==0
```
Finds port scans (SYN without ACK).

```
http.request.method == "POST" && http.content_length > 0
```
Candidate exfil or C&C POST requests.

```
dns.qry.name matches "[A-Za-z0-9+/=]{20,}"
```
Suspicious long subdomains in DNS (encoded data or tunneling).

For visual triage, adding Wireshark coloring rules for DNS TXT queries and HTTP POSTs makes them immediately visible in the packet list without needing to filter.

---

## Where It's At Right Now

Still very much a work in progress. The pipeline works fine, but the heuristics behind it are pretty basic right now, it catches the obvious stuff like beacon timing, large transfers, long DNS names, HTTP POST weirdness. What I actually want next is flow baselining, so it flags deviation from what's normal for a given network instead of just checking against fixed thresholds.

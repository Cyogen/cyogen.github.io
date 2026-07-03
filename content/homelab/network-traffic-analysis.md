+++
title = "Network Traffic Analysis: Tools, Techniques, and Practical Capture Work"
date = "2026-06-03T00:00:00-05:00"
tags = ["nta", "wireshark", "tcpdump", "tshark", "nmap", "network", "tls", "ftp", "bpf", "homelab"]
description = "Network traffic analysis from fundamentals to practical capture work covering Tcpdump, Wireshark, and BPF syntax for identifying anomalies and protocol behavior."
draft = false
+++

Network traffic analysis is one of those skills that ties everything else together, doesn't matter how good your SIEM detections are if you can't actually read what's on the wire when something looks off. These are my notes on the core tools, the syntax I keep forgetting and looking up, and the general approach I've picked up for spotting suspicious traffic.

## Capture Strategy Before Touching a Tool

Three things matter before the first packet is captured:

1. **Know your environment.** Baseline traffic first. You cannot identify anomalies if you do not know what normal looks like.
2. **Placement is everything.** Where the capture sensor sits determines what is visible. A tap on the wrong segment will miss lateral movement entirely.
3. **Persistence.** One capture tells you nothing. Pattern deviation is only visible over time.

## Analysis Approach

Start with standard protocols and clear them before digging into anything unusual. HTTP/S, FTP, email, and basic TCP/UDP will make up the bulk of traffic from most environments. Work through those first, then check standard remote access protocols: SSH, RDP, Telnet. Then look for what is left.

What to watch for:

- Periodic connections at regular intervals (beacons)
- Hosts contacting sites they have never contacted before
- Unusual ports being bound once or twice on a host
- Large outbound transfers (exfil candidates)
- Encrypted traffic to unexpected destinations

## BPF Syntax

Every major capture tool supports Berkeley Packet Filters. Learning BPF once covers Tcpdump, Wireshark display filters (similar syntax), Tshark, and most tap hardware.

```
host 192.168.1.100          # traffic to/from a specific host
port 443                     # traffic on a specific port
portrange 0-1024             # all well-known ports
less 64                      # packets smaller than 64 bytes
net 192.168.1.0/24          # traffic within a subnet
host X and port 23           # compound filter
```

## Tcpdump

Command-line packet capture. Available on every Unix-like system. WinDump (Windows port) is no longer maintained.

**List available interfaces:**

```bash
sudo tcpdump -D
```

**Capture on interface:**

```bash
sudo tcpdump -i eth0
```

**Include Ethernet header:**

```bash
sudo tcpdump -i eth0 -e
```

**Show ASCII and hex:**

```bash
sudo tcpdump -i eth0 -X
```

**Combined verbose capture:**

```bash
sudo tcpdump -i eth0 -nnvXX
```

- `-nn` disables hostname and port resolution (faster, shows raw IPs and numbers)
- `-vXX` verbose with hex and ASCII

**Practical filters:**

```bash
sudo tcpdump -i eth0 portrange 0-1024
sudo tcpdump -i eth0 less 64
sudo tcpdump -i eth0 host 192.168.0.1 and port 23
```

![Tcpdump verbose output with hex and ASCII](/images/posts/nta/tcpdump-output.png)

## Wireshark

Wireshark organizes each packet across three panes:

1. **Packet List (top):** one row per packet, summary fields (time, source, destination, protocol, info)
2. **Packet Details (bottom left):** full protocol breakdown, OSI model layered from bottom up (lowest layer at top of this pane)
3. **Packet Bytes (bottom right):** raw hex and ASCII; selecting a field in any other pane highlights its bytes here

Each row in the bytes pane shows the data offset, sixteen hex bytes, and sixteen ASCII bytes. Non-printable bytes appear as a period.

![Wireshark with packet list, details, and bytes panes](/images/posts/nta/wireshark-ui.png)

## Protocol Reference: TLS Handshake

A TLS session over HTTPS starts with a TCP connection on port 443, followed by a ClientHello to begin the handshake.

The handshake negotiates: session ID, peer x509 certificate, compression algorithm, cipher spec, whether the session is resumable, and the 48-byte master secret shared between client and server.

After the handshake completes, all payload data flows as TLS Application Data (opaque to passive capture). You can see that a connection exists and its size, but not the content.

![TLS handshake visible in Wireshark capture](/images/posts/nta/tls-handshake.png)

TLS handshake steps (RFC 2246):

1. Client and server exchange hello messages, agree on connection parameters
2. Exchange cryptographic parameters to establish premaster secret
3. Exchange x.509 certificates for authentication
4. Generate master secret from premaster secret and random values
5. Issue negotiated security parameters to the record layer
6. Both sides verify identical security parameters and that the handshake was not tampered with

## Protocol Reference: FTP

FTP traffic is cleartext. FTP commands in capture:

```
USER, PASS, PORT, PASV, LIST, CWD, PWD, SIZE, RETR, QUIT
```

![FTP request/response capture in Wireshark](/images/posts/nta/ftp-capture.png)

Requests from client to server, responses back. In Wireshark, follow the TCP stream on any FTP session to see credentials and file listings in plaintext. Any environment still running FTP for internal file transfer should flag this immediately.

## Triage Workflow

A quick workflow for analyzing an unknown PCAP:

1. Open summary output (top protocols, top talkers, top endpoints)
2. Review conversation logs sorted by bytes: persistent small flows suggest beaconing, large uploads suggest exfil
3. Review HTTP requests: inspect POST requests, look for base64 or hex-encoded bodies
4. Check DNS for long subdomains: encoded data tunneled through DNS will appear as unusually long query names
5. Extract files if the capture tool supports it: inspect any reconstructed binaries in a sandbox
6. Check IDS logs if Suricata or Snort ran against the capture

Wireshark display filters for rapid triage:

```
tcp.flags.syn==1 && tcp.flags.ack==0
```

This shows only SYN packets without ACK, which is every connection initiation. High volume from one source to many ports is a scan.

```
http.request.method == "POST" && http.content_length > 0
```

Candidate exfil: POST requests with a body. Inspect the content type and payload.

```
dns.qry.name matches "[A-Za-z0-9+/=]{20,}"
```

DNS names with long encoded-looking subdomains. DNS tunneling and C&C beaconing often use base64 or hex in the subdomain label.

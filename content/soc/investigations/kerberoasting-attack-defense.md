+++
title = "Kerberoasting: Attack Simulation and Detection in Active Directory"
date = "2026-06-01T00:00:00-05:00"
tags = ["active-directory", "kerberoasting", "kerberos", "hashcat", "detection", "homelab", "windows"]
description = "Simulating a Kerberoasting attack against an Active Directory environment using Rubeus and Hashcat, then building out detection and defensive controls."
draft = false
+++

Kerberoasting is one of those AD techniques that shows up constantly, it's usually one of the first things tried once someone's got any foothold at all. Part of why it's so popular is that it's quiet, needs no special privileges, and just abuses how Kerberos authentication is designed to work in the first place. I ran the full attack in the lab and then built out detection and defense around it.

## How Kerberoasting Works

Kerberos uses Service Principal Names (SPNs) to link service instances to their logon accounts. Any authenticated domain user can request a Ticket Granting Service (TGS) ticket for any SPN in the domain. The ticket comes back encrypted with the service account's NTLM hash.

Once that ticket's in hand you crack it completely offline, no more network noise, no lockout risk, and you never needed elevated privileges just to request it in the first place. That combination is exactly why it's dangerous.

## Environment

- Windows Server 2022 domain controller (DC01)
- Kali Linux attacker machine
- Rubeus (ticket extraction)
- Hashcat and John the Ripper (offline cracking)

## The Attack

### Step 1: Extract Tickets with Rubeus

From a domain-joined machine or a machine with valid domain credentials:

```
Rubeus.exe kerberoast /outfile:spn.txt
```

Rubeus queries the domain for all accounts with SPNs registered, requests a TGS ticket for each one, and dumps the hashes to a file. The output looks like a wall of Kerberos ticket data in hashcat-ready format.

![Rubeus output showing extracted TGS tickets](/images/posts/kerberoasting/rubeus-output.png)

### Step 2: Crack the Hash

Two options for cracking the extracted TGS hashes:

#### Hashcat

Mode 13100 handles Kerberoastable TGS tickets:

```bash
hashcat -m 13100 -a 0 spn.txt passwords.txt --outfile="cracked.txt"
```

If Hashcat returns a hardware error, add `--force`. Once finished, the cracked output shows the plaintext password alongside the hash.

![Hashcat cracked output](/images/posts/kerberoasting/hashcat-cracked.png)

#### John the Ripper

The same hashes crack with John using the `krb5tgs` format:

```bash
sudo john spn.txt --fork=4 --format=krb5tgs --wordlist=passwords.txt --pot=results.pot
```

## Detection

The primary detection signal is Event ID 4769 (Kerberos Service Ticket Operations). A Kerberoasting attempt will generate 4769 events with:

- Ticket Encryption Type: `0x17` (RC4-HMAC). Modern accounts use AES, so RC4 requests stand out
- Multiple 4769 events in a short window targeting different SPNs

![Event ID 4769 in Event Viewer](/images/posts/kerberoasting/event-4769.png)

The most reliable detection trap is a **honeypot service account**:

- Create an account that looks appealing (has privs, has an SPN registered, appears to have been around for 2+ years)
- Set a strong password (100+ characters) so it never actually cracks
- Alert on ANY 4769 targeting that account's SPN, successful or not

Any activity against a honeypot SPN is suspicious by definition. No legitimate service should be requesting tickets for an account that does not actually run a service.

## Defense

- Audit all SPNs in the domain and disable any that are no longer in use
- Service accounts must have long, randomly generated passwords (100+ characters minimum). Managed service accounts (gMSA) rotate automatically and are the better option
- Use Group Managed Service Accounts (gMSA) wherever possible to eliminate human-set passwords from the equation entirely
- Monitor for RC4 encryption type requests in 4769 events, especially in bulk

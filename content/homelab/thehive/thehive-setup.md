+++
title = "Setting Up TheHive for SOC Case Management"
date = "2026-06-25T14:00:00-05:00"
tags = ["thehive", "homelab", "case-management", "elasticsearch", "docker", "ubuntu", "soc"]
description = "Installing TheHive 3 on a System76 Lemur Pro: battling hardware quirks, a dead apt repository, and config format mismatches to get a working case management platform."
draft = false
+++

Wazuh throws alerts, but without something to turn them into actual trackable, assignable, closeable cases, a SIEM is just a wall of text nobody ever acts on. That's where TheHive comes in.

Getting it running on a System76 Lemur Pro turned into more of a fight than I expected. The hardware wasn't cooperating, the upstream repo I needed was just gone, and the docs I found pointed at the wrong config format entirely. Writing all of it down here so I don't have to rediscover any of this next time.

## Hardware

The System76 Lemur Pro (lemu8) is a thin ultrabook: 8GB RAM, no dedicated
GPU: running Ubuntu Server 26.04 minimal over WiFi.

Hostname: `thehive`. Static IP: `10.0.42.139`.

## Before the Install: Hardware Problems

Two issues had to be solved before touching any packages.

### Random Shutdowns at Idle

The machine was shutting down randomly with no load, no thermal event, and
nothing in the logs. This reproduced across multiple OS installs.

Root cause: the lid switch hardware is flaky. The kernel was reading spurious
closure events and triggering suspend: with the lid fully open.

```bash
sudo nano /etc/systemd/logind.conf
```

```ini
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
sudo systemctl kill -s HUP systemd-logind
```

Stable for 10+ minutes at idle immediately after. Problem closed.

### SSH Sessions Dropping

WiFi power management was putting the adapter to sleep mid-session. The fix
doesn't survive a reboot without a service to reapply it:

```bash
sudo nano /etc/systemd/system/wifi-powersave-off.service
```

```ini
[Unit]
Description=Disable WiFi power management
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/iw dev wlp1s0 set power_save off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now wifi-powersave-off
```

## The Stack

TheHive 3 requires:

- **Elasticsearch 7.x**: all TheHive data lives here
- **Docker**: TheHive runs containerized

Cassandra is a TheHive 4+ requirement. TheHive 3 doesn't use it. It's
installed here as a future upgrade path but plays no role in the current setup.

## Java: Version Conflicts

Ubuntu 26.04 defaults to Java 21. TheHive's Docker container is fine with
that. Cassandra 4.1.x is not: two JVM flags it depends on were removed in
newer versions:

- `UseBiasedLocking`: removed in Java 17
- `UseConcMarkSweepGC`: removed in Java 15

Only Java 11 works with Cassandra 4.1.x.

```bash
sudo apt-get install -y openjdk-21-jre-headless openjdk-11-jdk
```

Force Cassandra to use Java 11 in two places: the startup script and the
systemd unit: because one without the other doesn't hold:

```bash
# Add at line 1 of /etc/cassandra/cassandra-env.sh:
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```

```bash
sudo systemctl edit cassandra
```

```ini
[Service]
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now cassandra
nodetool status
# UN  127.0.0.1 : Up/Normal
```

## Elasticsearch

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/7.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt-get update && sudo apt-get install -y elasticsearch
```

`/etc/elasticsearch/elasticsearch.yml`:

```yaml
http.host: 127.0.0.1
transport.host: 127.0.0.1
cluster.name: hive
thread_pool.search.queue_size: 100000
path.logs: /var/log/elasticsearch
path.data: /var/lib/elasticsearch
xpack.security.enabled: false
script.allowed_types: inline
http.cors.enabled: true
```

```bash
sudo systemctl enable --now elasticsearch
```

### Gotcha: Duplicate Keys Crash Elasticsearch

The default `elasticsearch.yml` already defines `path.logs` and `path.data`.
Adding them again caused Elasticsearch to fail immediately on startup:

```
JsonParseException: Duplicate field 'path.logs' at line 102
```

Fix:

```bash
sudo sed -i '102,103d' /etc/elasticsearch/elasticsearch.yml
sudo systemctl start elasticsearch
```

Check for existing keys before appending to any config file.

## TheHive: The Repository That No Longer Exists

StrangeBee distributes TheHive 5 through `archives.strangebee.com`. As of
this writing, that domain is NXDOMAIN. Their Docker Hub images
(`strangebee/thehive`) show up in searches but have no public tags. The
official install docs are pointing at infrastructure that's gone.

```bash
nslookup archives.strangebee.com 8.8.8.8
# server can't find archives.strangebee.com: NXDOMAIN
```

The original open source image is still available:

```bash
docker pull thehiveproject/thehive:latest
# TheHive 3.5.2
```

TheHive 3 is fully open source. TheHive 5 moved to a commercial model under
StrangeBee. For a homelab SOC, 3.5.2 has everything needed.

## Installing Docker

```bash
sudo apt-get install -y docker.io docker-compose
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

## Configuring and Running TheHive

```bash
sudo mkdir -p /opt/thehive/{data,config}
```

`/opt/thehive/config/application.conf`:

```hocon
search {
  index: the_hive
  uri: "http://127.0.0.1:9200/"
}

storage {
  provider: localfs
  localfs.location: /opt/thp/thehive/data
}

play.http.secret.key: "your-secret-key-at-least-32-chars"
application.baseUrl = "http://10.0.42.139:9000"
```

**This is TheHive 3 format.** TheHive 4 and 5 use `db.janusgraph { ... }`.
Using the wrong format generates cryptic HOCON parse errors about unbalanced
braces: the parser won't tell you the version is the problem.

Run with host networking so the container can reach Elasticsearch on the
host's loopback:

```bash
docker run -d \
  --name thehive \
  --network host \
  -v /opt/thehive/config/application.conf:/etc/thehive/application.conf \
  -v /opt/thehive/data:/opt/thp/thehive/data \
  -e JVM_OPTS="-Xms512m -Xmx768m" \
  --restart unless-stopped \
  thehiveproject/thehive:latest
```

Verify:

```bash
curl -s http://127.0.0.1:9000/index.html | head -1
# <!doctype html>
```

Access at `http://10.0.42.139:9000`. Default credentials: `admin` / `secret`.
Change the password on first login.

TheHive's actually up and running now. Next step is wiring it to Wazuh so alerts turn into cases on their own, which I'll get into in the next post.

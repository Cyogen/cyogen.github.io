+++
title = "Investigating a Wazuh Rootcheck Alert: Trojaned chsh Binary"
date = "2026-06-25T20:00:00-05:00"
tags = ["wazuh", "soc", "false-positive", "rootcheck", "debian", "incident-response"]
description = "First alert from my Wazuh homelab: rootcheck flagged /usr/bin/chsh as a trojaned binary on a fresh Debian install. Walking through the investigation and resolution."
draft = false
+++

## The Alert

Shortly after deploying Wazuh on my Debian-based monitoring server, the
rootcheck module fired its first alert:

```
/usr/bin/chsh says it's a "Trojaned version of file detected"
```

Rootcheck is Wazuh's host-based anomaly detection engine. One of the things it
does is compare common system binaries against a database of known-good and
known-bad file signatures, flagging anything that looks tampered with.

A trojaned binary alert is serious: if real, it means an attacker has replaced
a legitimate system binary with a malicious one that runs attacker code while
appearing to function normally. `chsh` (change shell) is a common target because
it runs as setuid root.

## Investigation

**Step 1: Identify the owning package**

The first question: does the OS even claim to own this file?

```bash
dpkg -S /usr/bin/chsh
```

Output:
```
passwd: /usr/bin/chsh
```

`/usr/bin/chsh` is owned by the `passwd` package. Not installed manually, not
dropped by something unknown: a legitimate Debian package owns it.

**Step 2: Verify package integrity**

Knowing a package *claims* ownership isn't enough. The file itself could still
be modified. `dpkg -V` checks each file in a package against its stored checksum:

```bash
dpkg -V $(dpkg -S /usr/bin/chsh | cut -d: -f1)
```

Output:
```
(none)
```

No output means every file in the `passwd` package matches its expected checksum
exactly. The binary has not been modified.

## Conclusion

**False positive.** The binary is clean and verified by the package manager.

The root cause is that Wazuh's rootcheck signature database is built primarily
around Red Hat/CentOS binary signatures. On Debian (and Debian-derived systems),
the compiled binaries differ enough from what Wazuh expects that legitimate
system files trigger the trojaned binary check.

This is a known limitation of rootcheck on non-RHEL systems. It does not mean
rootcheck is useless: it would still catch a genuinely replaced binary: but
it generates noise on Debian that needs to be tuned out.

## Remediation: Suppressing the False Positive

To prevent this alert from firing again, create a custom suppression rule in
Wazuh. On the manager:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Add the following rule inside the `<group>` tags:

```xml
<group name="rootcheck,">
  <rule id="100001" level="0">
    <if_sid>510</if_sid>
    <match>chsh</match>
    <description>FP: /usr/bin/chsh verified clean by dpkg on Debian</description>
  </rule>
</group>
```

Setting `level="0"` suppresses the alert entirely. The `<if_sid>510</if_sid>`
targets the parent rootcheck rule, and `<match>chsh</match>` scopes it to only
this specific binary.

Restart the manager to apply:

```bash
sudo systemctl restart wazuh-manager
```

## Takeaway

Everyone doing this job runs into false positives constantly, that's just the reality of it. The point isn't to panic every time something fires, it's to actually check it out, land on a conclusion you can defend, and tune the noise down so the real stuff doesn't get buried. This whole thing took maybe two minutes once I knew where to look.

**Alert status:** Closed, false positive
**Rule tuned:** Yes, suppressed via local_rules.xml rule 100001

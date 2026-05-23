# 🔍 Project 4 — Vulnerability Scanning & Risk Assessment using Nessus

![Security](https://img.shields.io/badge/Security-SOC%20Analysis-red) ![Tool](https://img.shields.io/badge/Tool-Nessus%20Essentials-blue) ![Status](https://img.shields.io/badge/Status-Completed-brightgreen) ![Type](https://img.shields.io/badge/Type-Home%20Lab-orange)

---

## Why I Did This Project

Knowing a system has open ports is one thing. Knowing those ports are running vulnerable software with a working exploit available is another. That's the gap vulnerability scanning fills — it takes the raw network data from something like Nmap and turns it into actionable risk intelligence.

I wanted to get hands-on with Nessus because it's the industry standard. Most enterprise SOC environments and pentest engagements will have Nessus or a tool like it in the stack. Running it against both a hardened Windows 11 machine and a deliberately vulnerable Metasploitable VM gave me a good comparison — what a relatively secure system looks like versus a fully compromised one.

---

## Targets

| Target | IP | OS | Purpose |
|---|---|---|---|
| Windows Workstation | 192.168.100.4 | Windows 11 | Primary production-like machine |
| Metasploitable 2 | 192.168.211.128 | Ubuntu Linux 8.04 | Deliberately vulnerable lab target |

---

## Tool

**Nessus Essentials** — free tier, up to 16 IPs, runs as a web interface on `https://127.0.0.1:8834`. Scan type used was **Basic Network Scan** with CVSS v3.0 scoring.

---

## Part 1 — Windows 11 Workstation (192.168.100.4)

### Scan Summary

| Metric | Value |
|---|---|
| **Scan Duration** | 18 minutes |
| **Total Vulnerabilities** | 24 |
| **Critical** | 0 |
| **High** | 0 |
| **Medium** | 4 |
| **Low** | 0 |
| **Info** | 20 |

The Windows machine came back relatively clean — no Critical or High findings. The medium severity issues were mostly around SSL configuration and an outdated Splunk version. The info findings were largely service enumeration — Nessus confirming what services are running rather than flagging exploitable vulnerabilities.

---

### Key Findings — Windows Workstation

**Finding 1 — Splunk Enterprise Outdated (CVE-2026-20202)**
Severity: 🟡 Medium | CVSS: 6.6

The version of Splunk running on port 8000 was 10.2.1, and the patched version is 10.2.2. The vulnerability (SVD-2026-0401) allows a user with the `edit_user` capability to create a crafted username that causes inconsistencies in account management. Not remotely exploitable without authentication, but worth patching immediately given that Splunk is the security monitoring platform — the last thing you want is a vulnerability in your detection tooling.

There was a second Splunk finding — SVD-2026-0402, CVSS 4.3 — a lower severity variant of the same advisory.

*Fix: Upgrade Splunk Enterprise to 10.2.2 or later.*

---

**Finding 2 — SSL Certificate Cannot Be Trusted**
Severity: 🟡 Medium | CVSS: 6.5

The SSL certificate on port 8089 (Splunk management port) is self-signed — meaning it wasn't issued by a trusted certificate authority. Any client connecting to this port can't verify they're actually talking to the real Splunk server. In a network with a man-in-the-middle attacker, this makes interception significantly easier.

This is a common finding on internal services where people generate their own certs for convenience. Fine for a home lab but not acceptable in production.

*Fix: Replace the self-signed certificate with one from a trusted CA, or deploy an internal PKI.*

---

**Finding 3 — SSL Self-Signed Certificate**
Severity: 🟡 Medium | CVSS: 6.5

Related to Finding 2 — same root cause. Both SSL findings on the Windows machine point to the same issue: Splunk's HTTPS interface using certificates that can't be verified by external clients.

---

**Finding 4 — Microsoft Windows NTLMSSP Authentication Request Remote Network Name Disclosure**
Severity: ℹ️ Info

During authentication attempts, Windows leaks the machine name and domain via NTLM negotiation. Not directly exploitable on its own but useful for an attacker doing reconnaissance — it confirms the machine name and workgroup, which can feed into further attacks.

---

**Finding 5 — SMB Multiple Issues**
Severity: ℹ️ Info (5 findings)

Nessus enumerated the SMB stack in detail:
- SMB Service Detection
- NativeLanManager Remote System Information Disclosure — OS version and build leaked via SMB
- SMB Versions Supported — both SMB2 and SMB3 detected
- Windows NetBIOS/SMB Remote Host Information Disclosure

All informational on this machine, but worth noting that SMB is exposing system information to anyone on the network who asks for it.

---

### Windows Assessment Conclusion

The Windows 11 machine is in reasonable shape. The only real action items are patching Splunk to the latest version and sorting out the SSL certificates on the management interface. No critical exposure, no backdoors, no default credentials — just some housekeeping.

---

## Part 2 — Metasploitable 2 (192.168.211.128)

### Scan Summary

| Metric | Value |
|---|---|
| **Scan Duration** | 18 minutes |
| **Total Vulnerabilities** | 62 |
| **Critical** | 7 |
| **High** | 4 |
| **Medium** | 12 |
| **Low** | 6 |
| **Info** | 118 |

This is what a genuinely compromised system looks like. Metasploitable is designed to be vulnerable — it's a training target — but seeing it through Nessus makes it very real. Seven critical vulnerabilities, four of which Nessus was able to actively exploit during the scan. The donut chart on the dashboard was almost entirely red and orange.

---

### Key Findings — Metasploitable

**Finding 1 — Bind Shell Backdoor Detection**
Severity: 🔴 Critical | CVSS: 9.8

This is the most alarming finding. A shell is listening on port 1524 that requires no authentication whatsoever. Nessus connected to it and ran the command `id` — the output came back:

```
root@metasploitable:/# uid=0(root) gid=0(root) groups=0(root)
```

Anyone on the network can connect to port 1524 and immediately have a root shell on this machine. No exploit required. No credentials needed. Just connect and you own the system.

*Fix: Verify if system has been compromised. Reinstall if necessary. In production this finding would trigger an immediate incident response.*

---

**Finding 2 — VNC Server 'password' Password**
Severity: 🔴 Critical | CVSS: 10.0

The VNC server on port 5900 is secured with the password `password`. Nessus logged in using VNC authentication with that credential and confirmed full access. CVSS 10.0 — the maximum possible score.

VNC gives full graphical desktop access to the machine. Combined with the default password, this means any attacker on the network can take complete visual control of the system in seconds.

*Fix: Set a strong VNC password. Better — disable VNC if not required and use SSH instead.*

---

**Finding 3 — Debian OpenSSH/OpenSSL Package Random Number Generator Weakness**
Severity: 🔴 Critical | CVSS: 10.0

A well-known Debian bug (2008) where a packager accidentally removed almost all sources of entropy from the OpenSSL random number generator. This means any cryptographic keys generated on the affected system — SSH host keys, SSL certificates, VPN keys — have an extremely small keyspace. An attacker can enumerate all possible keys and brute-force SSH access.

Marked as "In the news: true" and "Exploit Available: true" in Nessus. This is a known, weaponized vulnerability.

*Fix: Regenerate all cryptographic keys. Upgrade OpenSSL and OpenSSH packages.*

---

**Finding 4 — Apache Tomcat AJP Connector Request Injection (Ghostcat)**
Severity: 🔴 Critical | CVSS: 9.8 | CVE: CVE-2020-1745

Ghostcat is a well-documented vulnerability in Apache Tomcat's AJP connector. A remote unauthenticated attacker can read any file from the web application, including configuration files containing credentials. If the application allows file uploads, it escalates to Remote Code Execution (RCE).

Nessus confirmed exploitation was possible — it was able to make a valid AJP request to the connector. EPSS score of 0.9447 means there's a 94.47% probability this vulnerability is being actively exploited in the wild.

*Fix: Upgrade Apache Tomcat to 7.0.100, 8.5.51, or 9.0.31 or later. Disable AJP connector if not needed.*

---

**Finding 5 — SSL Version 2 and 3 Protocol Detection**
Severity: 🔴 Critical | CVSS: 9.8

SSLv2 and SSLv3 are deprecated protocols with known cryptographic weaknesses. POODLE, DROWN, and BEAST attacks all target these protocol versions. Having them enabled means an attacker can potentially decrypt communications that should be encrypted.

*Fix: Disable SSLv2 and SSLv3. Enforce TLS 1.2 minimum, prefer TLS 1.3.*

---

**Finding 6 — Canonical Ubuntu Linux SEoL (8.04.x)**
Severity: 🔴 Critical | CVSS: 10.0

Ubuntu 8.04 reached end of life in May 2013. It has received no security patches for over a decade. Every vulnerability discovered since 2013 remains unpatched on this system. Running an EOL operating system in any environment — even a lab — is a critical risk because there's no remediation path other than upgrading.

*Fix: Upgrade to a supported Ubuntu release.*

---

**Finding 7 — NFS Shares World Readable**
Severity: 🔴 High | CVSS: 7.5

The NFS (Network File System) exports on this machine are configured to allow any host to mount them. This means anyone on the network can mount the file system and read its contents — potentially including sensitive files, credentials, and configuration data.

*Fix: Restrict NFS exports to specific trusted IP addresses in /etc/exports.*

---

**Finding 8 — Samba Badlock Vulnerability**
Severity: 🔴 High | CVSS: 7.5 | CVE: CVE-2016-2118

Badlock is a vulnerability in Samba's implementation of the Security Account Manager (SAM) and Local Security Authority (LSAD) protocols. A man-in-the-middle attacker can downgrade the authentication level and execute arbitrary Samba network calls in the context of the intercepted user — potentially gaining access to Active Directory data or disabling critical services.

*Fix: Upgrade Samba to version 4.2.11, 4.3.8, or 4.4.2 or later.*

---

**Finding 9 — ISC BIND Service Downgrade / Reflected DoS**
Severity: 🔴 High | CVSS: 8.6

Multiple vulnerabilities in the ISC BIND DNS server running on this machine. The most severe allows a remote attacker to cause a Denial of Service by sending specially crafted DNS queries. A reflected DoS attack using this vulnerability could amplify traffic and take down network services.

*Fix: Upgrade ISC BIND to version 9.11.22, 9.16.6, or 9.17.4 or later.*

---

### Remediations Tab

Nessus generated two priority remediation actions:
1. Upgrade ISC BIND to address 3 vulnerabilities
2. Upgrade Samba to address the Badlock vulnerability

The more severe findings — Bind Shell, VNC default password, EOL OS — don't have simple patch fixes. They require configuration changes, reinstallation, or OS upgrades.

---

### Metasploitable Assessment Conclusion

This machine is catastrophically vulnerable and would be completely owned within minutes on any network. The bind shell alone — requiring zero credentials and giving root access — means game over. The VNC default password means full desktop control. The EOL operating system means none of this gets better without a complete rebuild.

The value of scanning Metasploitable isn't learning how to fix it — it's understanding what real critical exposure looks like so you can recognize the severity indicators when you see them on a real system.

---

## Comparison — Windows vs Metasploitable

| Factor | Windows 11 | Metasploitable 2 |
|---|---|---|
| **Critical Vulns** | 0 | 7 |
| **High Vulns** | 0 | 4 |
| **Exploitable by Nessus** | No | Yes (root access achieved) |
| **Default Credentials** | No | Yes (VNC: "password") |
| **Backdoor Present** | No | Yes (port 1524) |
| **OS Support Status** | Supported | EOL since 2013 |
| **Overall Risk** | 🟡 Low-Medium | 🔴 Critical |

---

## What I Learned

Running these two scans back to back was genuinely valuable. The contrast makes the severity ratings meaningful — a CVSS 6.6 on the Windows machine feels very different after you've just seen CVSS 10.0 findings where Nessus literally logged in and ran commands as root.

A few things stood out:

- **CVSS scores matter but context matters more.** The Splunk vulnerability on the Windows machine is Medium severity, but it's on the security monitoring platform itself — that context makes it more urgent than the raw score suggests.

- **Default credentials are still devastating.** VNC with password "password" getting a CVSS 10.0 isn't dramatic scoring — it's accurate. Default credentials are consistently one of the most common initial access vectors in real breaches.

- **EOL software is a hidden risk.** Ubuntu 8.04 EOL gets a CVSS 10.0 not because there's a specific exploit, but because an EOL system has an infinite accumulation of unpatched vulnerabilities. It's not one problem — it's every problem discovered since 2013.

- **The Remediations tab is underrated.** Nessus prioritizing which patches to apply first is exactly the kind of output a SOC analyst would hand to a sysadmin during a remediation effort.

---

## Tools Used

| Tool | Purpose |
|---|---|
| **Nessus Essentials** | Vulnerability scanning and risk assessment |
| **Kali Linux** | Scanning platform |
| **Metasploitable 2** | Intentionally vulnerable lab target |
| **MITRE ATT&CK** | Threat classification |
| **CVE Database** | Vulnerability reference |

---

*Analyst: Mwaura | SOC Portfolio Project 4 | May 2026*


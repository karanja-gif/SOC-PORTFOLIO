# 🔐 Risk Assessment Report
## Network Enumeration — 192.168.100.0/24

---

| Field | Details |
|---|---|
| **Report ID** | RA-2026-002 |
| **Date** | April 4, 2026 |
| **Analyst** | Mwaura |
| **Scanning Platform** | Kali Linux |
| **Tool** | Nmap 7.98 |
| **Target Network** | 192.168.100.0/24 |
| **Primary Target** | 192.168.100.4 (Windows 11) |
| **Assessment Type** | Network Enumeration & Risk Assessment |
| **Status** | Completed |

---

## 1. Executive Summary

A network enumeration assessment was conducted on the home lab network `192.168.100.0/24` using Nmap 7.98 from a Kali Linux machine. The assessment revealed **126+ live hosts** and identified **8 open ports** on the primary Windows 11 workstation (192.168.100.4).

Key findings include an exposed SMB port (445) which represents a **critical risk**, two exposed NetBIOS/legacy ports, and two Splunk SIEM ports accessible from the local network. Immediate remediation is recommended for ports 445 and 139.

---

## 2. Scope

| Item | Details |
|---|---|
| **Network Scanned** | 192.168.100.0/24 |
| **Deep Scan Target** | 192.168.100.4 |
| **Scan Duration** | ~999 seconds (full version scan) |
| **Scan Method** | Black-box — no prior knowledge of services |
| **Authorization** | Authorized — analyst owns the network |

---

## 3. Methodology

The assessment followed a structured approach:

**Phase 1 — Discovery:** Host discovery scan (`-sn`) to map all live devices on the network. Identified 126+ active hosts including the gateway `dev.opt (192.168.100.1)`.

**Phase 2 — Enumeration:** SYN scan with version detection (`-sS -sV`) against primary target 192.168.100.4 to identify open ports and running services.

**Phase 3 — OS Detection:** OS fingerprinting (`-O`) to determine the target operating system.

**Phase 4 — Analysis:** Each open port was analyzed for associated vulnerabilities, mapped to MITRE ATT&CK techniques, and assigned a risk rating.

---

## 4. Findings Summary

| Port | Service | Risk | CVE Reference |
|---|---|---|---|
| 445/tcp | SMB | 🔴 Critical | CVE-2017-0144 (EternalBlue) |
| 139/tcp | NetBIOS | 🔴 High | LLMNR/NBT-NS Poisoning |
| 8000/tcp | Splunk HTTP | 🔴 High | Credential exposure risk |
| 8089/tcp | Splunk HTTPS | 🔴 High | REST API exposure |
| 135/tcp | MS RPC | 🟡 Medium | CVE-2003-0352 (historical) |
| 902/tcp | VMware Auth | 🟡 Medium | VMware authentication exposure |
| 912/tcp | VMware Auth | 🟡 Medium | VMware authentication exposure |
| 7070/tcp | RealServer SSL | 🟡 Medium | Unknown service exposure |

---

## 5. Detailed Findings

### Finding 1 — SMB Port 445 Open
**Risk:** 🔴 Critical
**Port:** 445/tcp
**Service:** microsoft-ds (SMB)

**Description:**
SMB (Server Message Block) is exposed on the network. This is the same protocol exploited by the WannaCry ransomware in 2017 via the EternalBlue vulnerability (MS17-010). Any device on the 192.168.100.0/24 network can attempt to connect to this port.

**Attack Scenarios:**
- EternalBlue exploitation if SMB is unpatched
- Pass-the-hash credential attacks
- Ransomware lateral movement
- Unauthorized file share access

**MITRE ATT&CK:** T1021.002 — SMB/Windows Admin Shares

**Remediation:**
```powershell
# Block SMB inbound via PowerShell
New-NetFirewallRule -DisplayName "Block SMB" -Direction Inbound -Protocol TCP -LocalPort 445 -Action Block
```

---

### Finding 2 — NetBIOS Port 139 Open
**Risk:** 🔴 High
**Port:** 139/tcp
**Service:** netbios-ssn

**Description:**
NetBIOS is a legacy protocol that exposes machine name, workgroup, and network information to any host on the network. It enables LLMNR/NBT-NS poisoning attacks where an attacker responds to broadcast queries to intercept credentials.

**Attack Scenarios:**
- LLMNR/NBT-NS poisoning using Responder tool
- NetBIOS name spoofing
- Credential harvesting via fake authentication

**MITRE ATT&CK:** T1557.001 — LLMNR/NBT-NS Poisoning and SMB Relay

**Remediation:**
Disable NetBIOS over TCP/IP via Network Adapter settings or Group Policy.

---

### Finding 3 — Splunk Ports 8000 & 8089 Network Accessible
**Risk:** 🔴 High
**Ports:** 8000/tcp (HTTP), 8089/tcp (HTTPS)
**Service:** Splunkd httpd

**Description:**
The Splunk SIEM platform is running and accessible from any device on the local network. While expected for a running Splunk instance, exposing the management API port (8089) to the full network increases the attack surface. An attacker on the network could:
- Attempt brute force against Splunk web interface
- Query the Splunk REST API on port 8089
- Access indexed security logs if authentication is bypassed

**Remediation:**
Bind Splunk to localhost only when not requiring network access:
```ini
# $SPLUNK_HOME/etc/system/local/web.conf
[settings]
server.socket_host = 127.0.0.1
```

---

### Finding 4 — VMware Authentication Ports 902 & 912
**Risk:** 🟡 Medium
**Ports:** 902/tcp, 912/tcp
**Service:** VMware Authentication Daemon

**Description:**
VMware authentication daemon ports are exposed, confirming a virtualization platform is running. These ports use VNC and SOAP protocols. If VMware software is outdated, authentication vulnerabilities could allow unauthorized access to virtual machines.

**Remediation:**
- Keep VMware tools updated
- Restrict ports 902/912 via firewall to localhost only

---

### Finding 5 — Microsoft RPC Port 135
**Risk:** 🟡 Medium
**Port:** 135/tcp
**Service:** msrpc

**Description:**
Microsoft RPC endpoint mapper is exposed. While required for many Windows services, RPC has historically been a target for remote exploitation. Should be restricted to trusted hosts.

---

## 6. Network Exposure Summary

```
192.168.100.0/24 Network
│
├── 192.168.100.1  (dev.opt — Gateway/Router)
├── 192.168.100.4  (Windows 11 — PRIMARY TARGET)
│   ├── 🔴 445  — SMB (Critical)
│   ├── 🔴 139  — NetBIOS (High)
│   ├── 🔴 8000 — Splunk HTTP (High)
│   ├── 🔴 8089 — Splunk HTTPS (High)
│   ├── 🟡 135  — MS RPC (Medium)
│   ├── 🟡 902  — VMware Auth (Medium)
│   ├── 🟡 912  — VMware Auth (Medium)
│   └── 🟡 7070 — RealServer (Medium)
└── 126+ other live hosts
```

---

## 7. Recommendations Summary

| Priority | Recommendation | Effort |
|---|---|---|
| 🔴 Immediate | Block SMB port 445 inbound via firewall | Low |
| 🔴 Immediate | Disable NetBIOS over TCP/IP | Low |
| 🔴 Immediate | Bind Splunk to localhost only | Low |
| 🟡 Short-term | Restrict VMware ports to localhost | Low |
| 🟡 Short-term | Restrict RPC to trusted hosts only | Medium |
| 🟢 Long-term | Implement network segmentation | High |
| 🟢 Long-term | Schedule regular Nmap scans | Low |
| 🟢 Long-term | Deploy IDS/IPS on network | High |

---

## 8. Lessons Learned

- A basic Nmap scan reveals a significant amount of information about a host — attackers use the same tools
- SMB and NetBIOS should always be reviewed first on Windows machines — they are consistently high-value targets
- Running services like Splunk create necessary but manageable exposure — binding to localhost reduces risk significantly
- 126+ hosts on a home network highlights the importance of network segmentation — IoT devices, phones, and workstations should be on separate VLANs
- OS detection via Nmap can be unreliable — always verify manually

---

## 9. References

- [MITRE ATT&CK T1021.002 — SMB](https://attack.mitre.org/techniques/T1021/002/)
- [MITRE ATT&CK T1557.001 — LLMNR Poisoning](https://attack.mitre.org/techniques/T1557/001/)
- [CVE-2017-0144 — EternalBlue](https://nvd.nist.gov/vuln/detail/CVE-2017-0144)
- [Nmap Documentation](https://nmap.org/docs.html)
- [Splunk Security Hardening Guide](https://docs.splunk.com/Documentation/Splunk/latest/Security/Hardeningstandards)

---

*Report prepared as part of SOC Analyst Portfolio — Home Lab Assessment*
*Analyst: Mwaura | April 4, 2026*

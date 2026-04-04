# 🌐 Project 2 — Network Enumeration and Risk Assessment using Nmap

![Security](https://img.shields.io/badge/Security-SOC%20Analysis-red) ![Tool](https://img.shields.io/badge/Tool-Nmap%207.98-green) ![Status](https://img.shields.io/badge/Status-Completed-brightgreen) ![Type](https://img.shields.io/badge/Type-Home%20Lab-blue)

---

## 🧠 Scenario

As part of a proactive security assessment, a network enumeration scan was conducted on a home lab network (`192.168.100.0/24`) using **Nmap 7.98** from a **Kali Linux** machine. The objective was to discover live hosts, identify open ports, enumerate running services, and assess the overall risk exposure of the network.

The scan identified **126+ live hosts** on the network and performed a deep service/version scan on the primary Windows workstation at **192.168.100.4**, revealing several high and medium risk findings.

---

## 🏗️ Environment

| Component | Details |
|---|---|
| **Scanning Machine** | Kali Linux |
| **Tool** | Nmap 7.98 |
| **Target Network** | 192.168.100.0/24 |
| **Primary Target** | 192.168.100.4 (Windows 11) |
| **Network Gateway** | dev.opt (192.168.100.1) |
| **Scan Type** | SYN Scan (-sS), Version Detection (-sV), OS Detection (-O), Host Discovery (-sn) |

---

## 🔍 Scans Performed

### 1. Network Host Discovery
```bash
nmap -sn 192.168.100.0/24
```
Discovers all live hosts on the network without port scanning.

### 2. SYN + Version Scan (Primary Target)
```bash
nmap -sS -sV 192.168.100.4
```
Identifies open ports and running service versions on the target host.

### 3. OS Detection
```bash
nmap -O 192.168.100.4
```
Attempts to fingerprint the operating system of the target.

### 4. Save Results to File
```bash
nmap -sS -sV -oN nmap_scan_results.txt 192.168.100.4
```

---

## 📊 Findings

### Network Discovery Results
| Metric | Value |
|---|---|
| **Network Range Scanned** | 192.168.100.0/24 |
| **Total Live Hosts Discovered** | 126+ |
| **Gateway Identified** | dev.opt (192.168.100.1) |
| **Primary Target** | 192.168.100.4 |

### Open Ports — 192.168.100.4

| Port | State | Service | Version | Risk |
|---|---|---|---|---|
| 135/tcp | Open | Microsoft RPC | Windows RPC | 🟡 Medium |
| 139/tcp | Open | NetBIOS-SSN | Windows NetBIOS | 🔴 High |
| 445/tcp | Open | SMB | Microsoft-DS | 🔴 Critical |
| 514/tcp | Filtered | Shell | Remote Shell | 🟡 Medium |
| 902/tcp | Open | VMware Auth | VMware Daemon 1.10 (VNC/SOAP) | 🟡 Medium |
| 912/tcp | Open | VMware Auth | VMware Daemon 1.0 (VNC/SOAP) | 🟡 Medium |
| 7070/tcp | Open | RealServer | SSL/RealServer | 🟡 Medium |
| 8000/tcp | Open | HTTP | Splunkd httpd | 🔴 High |
| 8089/tcp | Open | HTTPS | Splunkd httpd (SSL) | 🔴 High |

**Total Open Ports:** 8  
**Critical Findings:** 1 (SMB port 445)  
**High Risk Findings:** 3 (NetBIOS 139, Splunk 8000, Splunk 8089)

---

## 🚨 Risk Analysis

### 🔴 CRITICAL — Port 445 (SMB) Open

SMB (Server Message Block) on port 445 is one of the most dangerous exposed services. It was the attack vector used in the **WannaCry ransomware** (2017) and **EternalBlue** exploits. An attacker on the same network could:
- Exploit unpatched SMB vulnerabilities
- Perform pass-the-hash attacks
- Move laterally across the network

**MITRE ATT&CK:** T1021.002 — Remote Services: SMB/Windows Admin Shares

---

### 🔴 HIGH — Port 139 (NetBIOS) Open

NetBIOS is a legacy protocol that should be disabled on modern networks. It leaks:
- Machine name and workgroup information
- Network topology details
- Can be used for LLMNR/NBT-NS poisoning attacks

**MITRE ATT&CK:** T1557.001 — Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning

---

### 🔴 HIGH — Ports 8000 & 8089 (Splunk) Exposed

The Splunk SIEM is accessible on the local network on both HTTP (8000) and HTTPS management port (8089). While expected for a running SIEM, this exposure means:
- Any device on the network can attempt to access the Splunk web interface
- Brute force attacks against Splunk credentials are possible from the network
- Port 8089 (Splunk REST API) should be restricted to localhost only

---

### 🟡 MEDIUM — Ports 902 & 912 (VMware Authentication Daemon)

VMware authentication ports are exposed, indicating a virtualization platform is running. If left unpatched, VMware authentication daemons can be targeted via:
- Authentication bypass vulnerabilities
- VNC-based attacks

---

### 🟡 MEDIUM — Port 135 (Microsoft RPC)

RPC is required for many Windows services but has historically been vulnerable to remote exploitation. Should be restricted via firewall to trusted hosts only.

---

## 🛡️ Recommendations

### Immediate Actions

| Priority | Action | Port Affected |
|---|---|---|
| 🔴 Critical | Restrict SMB (445) to trusted hosts only via Windows Firewall | 445 |
| 🔴 Critical | Disable NetBIOS over TCP/IP if not needed | 139 |
| 🔴 High | Restrict Splunk ports to localhost (127.0.0.1) only | 8000, 8089 |
| 🟡 Medium | Firewall VMware ports from network access | 902, 912 |
| 🟡 Medium | Restrict RPC to internal trusted hosts | 135 |

### How to Disable NetBIOS (Windows 11)
```
Control Panel → Network Connections → Adapter Properties
→ TCP/IPv4 Properties → Advanced → WINS tab
→ Select "Disable NetBIOS over TCP/IP"
```

### How to Restrict Splunk to Localhost
Edit `$SPLUNK_HOME/etc/system/local/server.conf`:
```ini
[httpServer]
listenOnIPv4 = 127.0.0.1
```

### How to Restrict SMB via Windows Firewall
```powershell
New-NetFirewallRule -DisplayName "Block SMB Inbound" -Direction Inbound -Protocol TCP -LocalPort 445 -Action Block
```

### Long-Term Hardening
- Enable **Windows Defender Firewall** with strict inbound rules
- Conduct **regular Nmap scans** to detect new open ports
- Implement **network segmentation** — separate IoT devices from workstations
- Enable **SMB signing** to prevent man-in-the-middle attacks
- Keep all services **patched and updated**

---

## 📁 Repository Structure

```
Project-2-Network-Scanning/
├── README.md                  ← This file
├── nmap-commands.md           ← All Nmap commands used
├── risk-assessment-report.md  ← Formal risk assessment
└── screenshots/
    ├── 01-host-discovery.png
    ├── 02-service-scan.png
    ├── 03-os-detection.png
    └── 04-open-ports-summary.png
```

---

## 🧰 Tools Used

| Tool | Purpose |
|---|---|
| **Nmap 7.98** | Port scanning, service detection, OS fingerprinting |
| **Kali Linux** | Scanning platform |
| **MITRE ATT&CK** | Threat classification framework |

---

## 👤 Analyst

**Mwaura | SOC Analyst — Cybersecurity Portfolio**
*Home Lab Network Assessment | April 4, 2026*

---

> 💡 *This project is part of a cybersecurity portfolio demonstrating SOC analyst skills including network enumeration, service identification, risk assessment, and remediation planning.*

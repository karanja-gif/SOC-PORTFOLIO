,# 🔐 Project 1 — Detecting and Responding to Brute Force Attacks using Splunk

![Security](https://img.shields.io/badge/Security-SOC%20Analysis-red) ![SIEM](https://img.shields.io/badge/SIEM-Splunk-green) ![Status](https://img.shields.io/badge/Status-Completed-brightgreen) ![Type](https://img.shields.io/badge/Type-Lab%20Simulation-blue)

---

## 🧠 Scenario

On **March 30, 2026**, suspicious authentication activity was detected on workstation **DESKTOP-58QET86**. Windows Security Event logs ingested into Splunk revealed a pattern consistent with a **brute force login attack** targeting the local user account **MWAURA**.

The attack was simulated in a controlled lab environment on a personal machine running **Windows OS** to demonstrate real-world SOC detection and response capabilities.

---

## 🏗️ Environment

| Component | Details |
|---|---|
| **SIEM Platform** | Splunk (local instance) |
| **Target Machine** | DESKTOP-58QET86 |
| **Operating System** | Windows (Workgroup environment) |
| **Log Source** | Windows Security Event Log |
| **Simulation Type** | Local brute force via interactive logon (Logon Type 2) |

---

## 🔍 Detection — Splunk Query

### Primary Detection Query
```spl
index=main (EventCode=4625 OR EventCode=4624)
```

### Refined Brute Force Query (5+ failures within short timeframe)
```spl
index=main EventCode=4625
| stats count by Account_Name, Source_Network_Address, host
| where count >= 5
| sort - count
```

### Timeline Query (attack pattern visibility)
```spl
index=main EventCode=4625 Account_Name=MWAURA
| timechart span=5s count
```

---

## 📊 Findings

### Failed Login Attempts (EventCode 4625)

| # | Timestamp | Account | Source IP | Logon Type | Failure Reason |
|---|---|---|---|---|---|
| 1 | 2026-03-30 05:43:30 | MWAURA | 127.0.0.1 | 2 (Interactive) | Bad password |
| 2 | 2026-03-30 05:43:34 | MWAURA | 127.0.0.1 | 2 (Interactive) | Bad password |
| 3 | 2026-03-30 05:43:39 | MWAURA | 127.0.0.1 | 2 (Interactive) | Bad password |
| 4 | 2026-03-30 05:43:43 | MWAURA | 127.0.0.1 | 2 (Interactive) | Bad password |
| 5 | 2026-03-30 05:43:47 | MWAURA | 127.0.0.1 | 2 (Interactive) | Bad password |
| 6 | 2026-03-30 05:43:52 | MWAURA | 127.0.0.1 | 2 (Interactive) | Bad password |

> ⚠️ **6 failed attempts in 22 seconds** — a clear indicator of automated or rapid manual brute force activity.

### Successful / Privileged Logon Events

| EventCode | Timestamp | Account | Logon Type | Notes |
|---|---|---|---|---|
| 4624 | 2026-03-30 05:44:44 | SYSTEM | 5 (Service) | Service logon post-attack |
| 4627 | 2026-03-30 05:44:44 | SYSTEM | 5 (Service) | Group membership logged |
| 4672 | 2026-03-30 05:44:44 | SYSTEM | — | **Special privileges assigned** |

> 🚨 **EventCode 4672** indicates special privileges (SeDebugPrivilege, SeImpersonatePrivilege, etc.) were assigned — a high-severity event requiring investigation when correlated with prior brute force activity.

### Attack Timeline

```
05:43:30 ─── Attempt 1 (FAIL)
05:43:34 ─── Attempt 2 (FAIL)  ← ~4s gap
05:43:39 ─── Attempt 3 (FAIL)  ← ~5s gap
05:43:43 ─── Attempt 4 (FAIL)  ← ~4s gap
05:43:47 ─── Attempt 5 (FAIL)  ← ~4s gap
05:43:52 ─── Attempt 6 (FAIL)  ← ~5s gap
              [52 second gap]
05:44:44 ─── SYSTEM logon + Special Privileges Assigned ⚠️
```

---

## 🚨 Incident Response

### Severity Assessment
| Factor | Value |
|---|---|
| **Severity** | HIGH |
| **MITRE ATT&CK Technique** | T1110 — Brute Force |
| **MITRE ATT&CK Sub-technique** | T1110.001 — Password Guessing |
| **Affected Account** | MWAURA |
| **Affected Host** | DESKTOP-58QET86 |

### Containment Actions Taken
- [x] Identified the targeted account (MWAURA) via Splunk correlation
- [x] Reviewed subsequent privileged logon (EventCode 4672) for signs of compromise
- [x] Isolated the event timeline to determine attack window
- [x] Confirmed source IP (127.0.0.1) as local — no lateral movement detected
- [x] Documented all IOCs (Indicators of Compromise)

### Indicators of Compromise (IOCs)
```
Account Targeted : MWAURA
Source IP        : 127.0.0.1
Hostname         : DESKTOP-58QET86
Attack Window    : 05:43:30 – 05:43:52 UTC
Failed Count     : 6 in 22 seconds
EventCodes       : 4625, 4624, 4627, 4672
Process          : C:\Windows\System32\svchost.exe
```

---

## 🛡️ Recommendations

### 1. Enable Multi-Factor Authentication (MFA)
Even if a password is compromised via brute force, MFA prevents unauthorized access by requiring a second verification factor.

### 2. Implement Account Lockout Policy
Configure Group Policy to lock accounts after **3–5 failed attempts** within a defined time window.
```
Computer Configuration → Windows Settings → Security Settings
→ Account Policies → Account Lockout Policy
  - Account lockout threshold: 5 attempts
  - Lockout duration: 30 minutes
  - Reset counter after: 15 minutes
```

### 3. Deploy Splunk Alert for Brute Force Detection
```spl
index=main EventCode=4625
| stats count by Account_Name, Source_Network_Address
| where count > 5
| eval alert="BRUTE FORCE DETECTED"
```
Set this as a **scheduled alert** (every 5 minutes) with email/SIEM notification.

### 4. Restrict Interactive Logons
Limit who can log on locally via Group Policy — especially on sensitive machines.

### 5. Monitor EventCode 4672
Always correlate **Special Privileges Assigned** events with preceding authentication events. Privilege assignment after failed logins is a strong post-compromise indicator.

---

## 📁 Repository Structure

```
Project-1-Brute-Force-Detection/
├── README.md                  ← This file (full documentation)
├── splunk-queries.md          ← All Splunk SPL queries used
├── incident-report.md         ← Formal SOC incident report
└── screenshots/               ← Add your Splunk dashboard screenshots here
    ├── splunk-search-results.png
    ├── failed-logins-chart.png
    └── timeline-visualization.png
```

---

## 🧰 Tools Used

- **Splunk** — Log ingestion, search, and visualization
- **Windows Event Viewer** — Raw log source
- **MITRE ATT&CK Framework** — Threat classification

---

## 👤 Analyst

**SOC Analyst | Cybersecurity Portfolio Project**  
*Simulated lab environment — DESKTOP-58QET86 | March 30, 2026*

---

> 💡 *This project is part of a cybersecurity portfolio demonstrating SOC analyst skills including log analysis, threat detection, incident response and SIEM usage.*

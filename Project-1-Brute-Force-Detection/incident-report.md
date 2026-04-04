# 🚨 SOC Incident Report
## Brute Force Attack — DESKTOP-58QET86

---

| Field | Details |
|---|---|
| **Incident ID** | INC-2026-001 |
| **Date Detected** | March 30, 2026 |
| **Time Detected** | 05:43:30 AM |
| **Severity** | 🔴 HIGH |
| **Status** | Resolved (Lab Simulation) |
| **Analyst** | SOC Analyst — Cybersecurity Portfolio |
| **SIEM Platform** | Splunk |

---

## 1. Executive Summary

On March 30, 2026, Splunk detected a brute force attack targeting the local user account **MWAURA** on workstation **DESKTOP-58QET86**. Six consecutive failed login attempts (EventCode 4625) occurred within a 22-second window between 05:43:30 and 05:43:52 AM, consistent with automated or rapid manual credential guessing.

The attack was followed approximately 52 seconds later by a SYSTEM-level logon (EventCode 4624) and the assignment of special privileges (EventCode 4672), which required further investigation to rule out post-compromise activity.

This incident was conducted as a **controlled lab simulation** to demonstrate SOC detection and response methodology.

---

## 2. Affected Assets

| Asset | Details |
|---|---|
| **Hostname** | DESKTOP-58QET86 |
| **Domain** | WORKGROUP |
| **OS** | Windows |
| **Targeted Account** | MWAURA |
| **Source IP** | 127.0.0.1 (localhost) |

---

## 3. Attack Timeline

| Time | EventCode | Description |
|---|---|---|
| 05:43:30 AM | 4625 | Failed login attempt #1 — MWAURA |
| 05:43:34 AM | 4625 | Failed login attempt #2 — MWAURA |
| 05:43:39 AM | 4625 | Failed login attempt #3 — MWAURA |
| 05:43:43 AM | 4625 | Failed login attempt #4 — MWAURA |
| 05:43:47 AM | 4625 | Failed login attempt #5 — MWAURA |
| 05:43:52 AM | 4625 | Failed login attempt #6 — MWAURA |
| 05:44:44 AM | 4624 | SYSTEM account logon (Logon Type 5 — Service) |
| 05:44:44 AM | 4627 | Group membership logged for SYSTEM |
| 05:44:44 AM | 4672 | ⚠️ Special privileges assigned to SYSTEM |

**Total failed attempts:** 6  
**Attack duration:** 22 seconds  
**Average interval between attempts:** ~4.4 seconds

---

## 4. Technical Analysis

### 4.1 Brute Force Pattern
The 6 failed logon attempts against account MWAURA occurred at highly regular intervals (~4 seconds apart), indicating a scripted or tool-assisted brute force approach rather than accidental mistyping.

**Logon Type 2** (Interactive) confirms the attempts were made directly at the machine console — not over the network.

### 4.2 Failure Codes
- **Status 0xC000006D** — The username or authentication information was invalid
- **Sub Status 0xC000006A** — Specifically indicates a wrong password (username was valid)

> 🔴 Sub Status 0xC000006A confirms that the account **MWAURA exists** on the system — the attacker was targeting a valid account with password guessing.

### 4.3 Post-Attack Privileged Activity
At 05:44:44 AM — 52 seconds after the last failed attempt — EventCodes 4624, 4627, and 4672 were logged for the **SYSTEM** account via `services.exe` (Logon Type 5).

While this is consistent with normal Windows service startup behavior, the correlation with preceding brute force activity warrants investigation. Key privileges assigned included:
- `SeDebugPrivilege`
- `SeImpersonatePrivilege`
- `SeTcbPrivilege`
- `SeSecurityPrivilege`

**Conclusion:** In this lab context, the SYSTEM logon was a normal Windows service event, not a result of the brute force. However, this correlation pattern should always be treated as suspicious in a production environment.

---

## 5. Detection Method

**Tool:** Splunk SIEM  
**Query Used:**
```spl
index=main (EventCode=4625 OR EventCode=4624)
```

**Trigger:** 6 failed logins (EventCode 4625) against account MWAURA detected within 22 seconds — exceeds brute force threshold of 5 failures.

---

## 6. Containment & Response Actions

| Action | Status |
|---|---|
| Identified targeted account (MWAURA) | ✅ Completed |
| Correlated failed logins with privileged events | ✅ Completed |
| Reviewed EventCode 4672 for compromise indicators | ✅ Completed |
| Confirmed source as local (127.0.0.1) — no network spread | ✅ Completed |
| Documented all IOCs | ✅ Completed |
| No lateral movement detected | ✅ Confirmed |

---

## 7. Indicators of Compromise (IOCs)

```
Type             : Brute Force / Credential Attack
Account          : MWAURA
Source IP        : 127.0.0.1
Hostname         : DESKTOP-58QET86
Attack Start     : 2026-03-30 05:43:30 AM
Attack End       : 2026-03-30 05:43:52 AM
Failed Count     : 6
EventCodes       : 4625, 4624, 4627, 4672
Failure Status   : 0xC000006D / 0xC000006A
Caller Process   : C:\Windows\System32\svchost.exe
MITRE Technique  : T1110 — Brute Force
MITRE Sub-tech   : T1110.001 — Password Guessing
```

---

## 8. Recommendations

### Immediate Actions
1. **Enable Account Lockout Policy** — Lock accounts after 5 failed attempts within 10 minutes
2. **Enable MFA** — Require a second factor for all interactive logons
3. **Create Splunk Alert** — Automate detection for future brute force attempts

### Long-Term Hardening
4. **Restrict Interactive Logons** — Limit Logon Type 2 to authorized users only via Group Policy
5. **Audit Privileged Events** — Always investigate EventCode 4672 when correlated with 4625
6. **Password Policy** — Enforce minimum 12-character complex passwords
7. **Baseline Normal Behavior** — Document expected SYSTEM service logon patterns to reduce false positives

---

## 9. Lessons Learned

- Sub Status code **0xC000006A** is a powerful indicator — it confirms the username is valid, narrowing the attack to password guessing
- Regular interval between attempts (~4s) distinguishes automated attacks from human mistyping
- EventCode **4672** should always be reviewed in context — benign in isolation, high-risk when correlated with brute force activity
- Splunk's `timechart` function is effective for visually identifying rapid attack bursts

---

## 10. References

- [MITRE ATT&CK T1110 — Brute Force](https://attack.mitre.org/techniques/T1110/)
- [Microsoft EventCode 4625 Documentation](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4625)
- [Microsoft EventCode 4672 Documentation](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4672)
- [Splunk SPL Documentation](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/WhatsInThisManual)

---

*Report generated as part of SOC Analyst Portfolio — Lab Simulation Environment*  
*Date: March 30, 2026 | Analyst: MWAURA*

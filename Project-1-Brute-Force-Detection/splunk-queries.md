# 🔍 Splunk Queries — Brute Force Detection

All SPL (Search Processing Language) queries used in this project.

---

## 1. Basic Detection Query
Detects all failed and successful logon events.
```spl
index=main (EventCode=4625 OR EventCode=4624)
```

---

## 2. Failed Logins Only
```spl
index=main EventCode=4625
| table _time, Account_Name, Source_Network_Address, host, Failure_Reason
| sort _time
```

---

## 3. Brute Force Threshold Detection
Flags accounts with 5 or more failures — core brute force indicator.
```spl
index=main EventCode=4625
| stats count by Account_Name, Source_Network_Address, host
| where count >= 5
| sort - count
| rename count as "Failed Attempts"
```

---

## 4. Attack Timeline Visualization
Plots failed logins over time to reveal rapid-fire attempt patterns.
```spl
index=main EventCode=4625 Account_Name=MWAURA
| timechart span=5s count as "Failed Logins"
```

---

## 5. Correlate Failures with Successful Logins
Shows if any account succeeded after repeated failures (potential compromise).
```spl
index=main (EventCode=4625 OR EventCode=4624)
| eval status=if(EventCode=4625,"FAILED","SUCCESS")
| stats count by Account_Name, status, Source_Network_Address
| sort Account_Name
```

---

## 6. Privileged Logon Alert (EventCode 4672)
Detects special privilege assignment — critical post-compromise indicator.
```spl
index=main EventCode=4672
| table _time, Account_Name, Privileges, host
| sort _time
```

---

## 7. Full Attack Correlation Query
Correlates brute force failures with subsequent privileged logons.
```spl
index=main (EventCode=4625 OR EventCode=4624 OR EventCode=4672)
| eval event_type=case(
    EventCode=4625, "FAILED LOGIN",
    EventCode=4624, "SUCCESSFUL LOGIN",
    EventCode=4672, "PRIVILEGE ASSIGNED"
  )
| table _time, EventCode, event_type, Account_Name, Source_Network_Address, host
| sort _time
```

---

## 8. Scheduled Alert Query (Set to run every 5 minutes)
Deploy this as a Splunk saved alert with email notification.
```spl
index=main EventCode=4625 earliest=-5m
| stats count by Account_Name, Source_Network_Address, host
| where count >= 3
| eval severity="HIGH"
| eval alert_message="BRUTE FORCE ATTACK DETECTED on " + host
```

---

## 📌 Event Codes Reference

| EventCode | Description | Severity |
|---|---|---|
| 4625 | An account failed to log on | Medium–High |
| 4624 | An account was successfully logged on | Low (monitor in context) |
| 4627 | Group membership information logged | Low |
| 4672 | Special privileges assigned to new logon | HIGH |

---

## 📌 Status / SubStatus Codes Found in Logs

| Status Code | Meaning |
|---|---|
| 0xC000006D | Unknown username or bad password |
| 0xC000006A | Bad password specifically |

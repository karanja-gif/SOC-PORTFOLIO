# 🔍 Nmap Commands — Network Enumeration & Risk Assessment

All commands used during this network assessment.

---

## 1. Network Host Discovery
Discovers all live hosts without port scanning. Fast and stealthy.
```bash
nmap -sn 192.168.100.0/24
```
**Result:** 126+ live hosts discovered on the network.

---

## 2. SYN Scan + Version Detection
Half-open SYN scan (stealthy) combined with service version detection.
```bash
nmap -sS -sV 192.168.100.4
```
**Result:** 8 open ports identified with service versions.

---

## 3. OS Fingerprinting
Attempts to identify the operating system of the target.
```bash
nmap -O 192.168.100.4
```
**Result:** Windows OS detected (Windows 11 confirmed manually).

---

## 4. Full Combined Scan
Runs SYN scan, version detection and OS detection together.
```bash
nmap -sS -sV -O 192.168.100.4
```

---

## 5. Save Results to File
Saves scan output to a text file for documentation.
```bash
nmap -sS -sV -oN nmap_scan_results.txt 192.168.100.4
```

---

## 6. Aggressive Scan (All info at once)
```bash
nmap -A 192.168.100.4
```
⚠️ Noisy — use only on networks you own.

---

## 📌 Nmap Flag Reference

| Flag | Meaning |
|---|---|
| `-sS` | SYN scan (half-open, stealthy) |
| `-sV` | Service/version detection |
| `-sn` | Host discovery only (no port scan) |
| `-O` | OS fingerprinting |
| `-A` | Aggressive (OS + version + scripts + traceroute) |
| `-p-` | Scan all 65535 ports |
| `-oN` | Save output in normal format |
| `-oX` | Save output in XML format |

---

## 📌 Port States Reference

| State | Meaning |
|---|---|
| `open` | Port is actively accepting connections |
| `closed` | Port is accessible but no service listening |
| `filtered` | Firewall is blocking the port — state unknown |
| `open\|filtered` | Cannot determine if open or filtered |

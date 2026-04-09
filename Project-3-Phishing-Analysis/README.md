# 📧 Project 3 — Phishing Email Analysis & Email Spoofing Demonstration

![Security](https://img.shields.io/badge/Security-SOC%20Analysis-red) ![Tool](https://img.shields.io/badge/Tool-swaks%20%7C%20sendmail-green) ![Status](https://img.shields.io/badge/Status-Completed-brightgreen) ![Type](https://img.shields.io/badge/Type-Home%20Lab-blue)

---

## Why I Did This Project

Phishing is the number one way attackers get into organizations. I wanted to understand it from both sides — what does a spoofed email actually look like under the hood, and how does an attacker send one? This project covers the analysis side and the attack simulation side.

The spoofing demo specifically came from something I kept seeing — attackers sending emails that look like they're from a CEO or finance director but they never actually logged into that account. I wanted to understand exactly how that works technically.

---

## What This Project Covers

- Analyzing a CEO fraud phishing email and breaking down every suspicious indicator
- Simulating email spoofing using swaks and sendmail on Kali Linux
- Understanding SPF, DKIM and DMARC — what they are and why they matter
- Documenting what happens when spoofed email hits a hardened mail server vs an unprotected one

---

## Environment

| Component | Details |
|---|---|
| **Attack Machine** | Kali Linux |
| **Tools Used** | swaks, sendmail 8.18.2 |
| **Scenario** | CEO Fraud / Wire Transfer Phishing |
| **Target (lab only)** | Analyst's own email accounts |
| **Spoofed Sender** | ceo@acmecorp.com (fictional company) |

---

## Part 1 — Phishing Email Analysis

### The Email

Below is the phishing email used in this simulation. This is a classic CEO fraud template — urgent, confidential, financial.

```
From: ceo@acmecorp.com
Reply-To: attacker.finance@protonmail.com
To: finance@acmecorp.com
Subject: Urgent Wire Transfer Required
Date: Tue, 07 Apr 2026 23:11:58 -0400

Dear Finance Team,

Please process an urgent wire transfer of $50,000 to the following 
account immediately. This is strictly confidential — do not discuss 
with anyone.

Bank: First National Bank
Account No: 1234567890
Routing No: 021000021

Time sensitive — must be done before close of business today.

Regards,
James Mitchell
CEO, AcmeCorp Ltd
```

---

### Red Flags — What I Noticed

**1. Reply-To Mismatch**
The From address is `ceo@acmecorp.com` but the Reply-To is `attacker.finance@protonmail.com`. This is the most telling sign. If the finance team replies, the response goes to the attacker — not the CEO. Real internal emails don't route replies to ProtonMail.

**2. Urgency + Secrecy Combined**
"Time sensitive" and "do not discuss with anyone" in the same email is a deliberate psychological tactic. It's designed to stop the recipient from pausing to verify. Any legitimate financial request should go through proper approval channels — the urgency is manufactured pressure.

**3. No Digital Signature**
Legitimate executive emails in most organizations use email signing or at minimum come through the company mail server. This email has no DKIM signature, no organizational branding, nothing that ties it to a real infrastructure.

**4. Specific Bank Details in Email Body**
Real wire transfer requests don't arrive via email with full banking details in the body. Finance teams use dedicated secure payment systems or face-to-face/phone verification for anything above a certain threshold.

**5. Sending Infrastructure Mismatch**
Looking at the raw headers (see screenshots), the email originated from `kali` via `127.0.0.1` — a localhost — not from any acmecorp.com mail server. The X-Mailer header also reveals swaks was used, which no legitimate mail client would show.

---

### Header Analysis

When you click "Show Original" or "View Raw Message" on any email client, you see the full headers. Here's what the headers on this spoofed email revealed:

```
Received: from kali (localhost [127.0.0.1])
Message-Id: <20260407231158.009335@kali>
X-Mailer: swaks v20240103.0 jetmore.org/john/code/swaks/
From: ceo@acmecorp.com
Reply-To: attacker.finance@protonmail.com
```

**Key observations:**
- `Received: from kali` — the email came from a machine named "kali", not from acmecorp.com's mail servers
- `X-Mailer: swaks` — this is a penetration testing tool, not Outlook or Gmail
- No `DKIM-Signature` header present — meaning the sending domain never signed this email
- SPF would fail — acmecorp.com has no SPF record authorizing kali/127.0.0.1 to send on its behalf

---

## Part 2 — Email Spoofing Demonstration

### How Email Spoofing Works

SMTP (Simple Mail Transfer Protocol) — the protocol that sends email — was designed in 1982 and has no built-in authentication. Anyone who can connect to an SMTP server can put anything in the `MAIL FROM` and `From:` fields. That's the core vulnerability.

Modern defenses (SPF, DKIM, DMARC) were added later to compensate, but many organizations still haven't properly configured them — making spoofing attacks viable against those targets.

### The Three Defenses (And How Attackers Bypass Them)

**SPF (Sender Policy Framework)**
A DNS record that says "only these IP addresses are allowed to send email for our domain." If an email comes from an unauthorized IP, receiving servers can reject it.

*Bypass:* Attack domains that have no SPF record, or misconfigured ones with `~all` (soft fail) instead of `-all` (hard fail).

**DKIM (DomainKeys Identified Mail)**
A cryptographic signature added to emails that proves the email was sent by the domain's actual mail server and wasn't tampered with in transit.

*Bypass:* Domains without DKIM configured can't sign emails, so receivers can't verify authenticity.

**DMARC (Domain-based Message Authentication)**
Builds on SPF and DKIM — tells receiving servers what to do when checks fail (reject, quarantine, or do nothing). Also sends reports back to the domain owner.

*Bypass:* Organizations with `p=none` DMARC policy get reports but emails still deliver even when SPF/DKIM fail.

---

### Lab Demonstration

**Setup:**
```bash
# Install sendmail
sudo apt install sendmail -y

# Start the service
sudo service sendmail start
```

**Spoofed email sent via swaks:**
```bash
swaks --to mwaura090@gmail.com \
      --from ceo@acmecorp.com \
      --header "Subject: Urgent Wire Transfer Required" \
      --header "Reply-To: attacker.finance@protonmail.com" \
      --body "Dear Finance Team,

Please process an urgent wire transfer of $50,000 immediately.
This is strictly confidential.

Regards,
James Mitchell
CEO, AcmeCorp Ltd" \
      --server 127.0.0.1 \
      --port 25
```

**SMTP Handshake Result:**
```
=== Connected to 127.0.0.1
<-  220 kali ESMTP Sendmail 8.18.2
-> MAIL FROM:<ceo@acmecorp.com>
<-  250 2.1.0 <ceo@acmecorp.com>... Sender ok
-> RCPT TO:<mwaura090@gmail.com>
<-  250 2.1.5 <mwaura090@gmail.com>... Recipient ok
<-  250 2.0.0 Message accepted for delivery
```

The local sendmail server accepted the spoofed sender without question — exactly as expected given SMTP has no native authentication.

---

### What Happened Next — Gmail's Response

The email never arrived in the inbox or spam folder. Gmail silently rejected it.

This is actually the correct behavior and demonstrates Gmail's security stack working as intended:

- **SPF check:** acmecorp.com has no SPF record authorizing 127.0.0.1 → FAIL
- **DKIM check:** No DKIM signature present → FAIL  
- **DMARC check:** No DMARC policy for acmecorp.com → no enforcement, but Gmail's own filters still rejected it based on reputation and SPF/DKIM failures

Gmail goes beyond just DMARC enforcement — it uses its own reputation systems to silently drop mail that looks suspicious, even when DMARC policy is missing.

---

### Why This Still Matters

Gmail rejecting the email might seem like "the attack failed" but that's only half the picture. The same attack against:

- A company running their own on-premise mail server with no SPF/DMARC → **email delivers to inbox**
- A company with SPF `~all` (soft fail) instead of `-all` → **email may still deliver**
- A company with DMARC `p=none` → **email delivers and owner just gets a report**
- An employee using a personal email for work → **likely delivers**

This is why phishing via spoofing is still one of the most used initial access techniques despite being a decades-old attack.

---

## MITRE ATT&CK Mapping

| Technique | ID | Description |
|---|---|---|
| Phishing | T1566 | Sending fraudulent emails to targets |
| Spearphishing via Service | T1566.003 | CEO fraud targeting specific individuals |
| Internal Spearphishing | T1534 | Impersonating internal executives |

---

## Detection — What SOC Analysts Should Look For

When reviewing email logs or investigating a suspected phishing email, these are the indicators I look for:

1. **Reply-To differs from From** — almost always malicious
2. **Received header doesn't match claimed sender domain** — email routing lies
3. **SPF/DKIM/DMARC failures in headers** — authentication breakdown
4. **X-Mailer showing unusual tools** — swaks, curl, python scripts
5. **Urgency language + financial request** — social engineering pattern
6. **Protonmail/Tutanota Reply-To on corporate email** — attacker hiding real address

---

## Recommendations

**For Organizations:**
- Configure SPF with `-all` hard fail — not `~all`
- Implement DKIM signing on all outbound mail
- Set DMARC to `p=reject` after monitoring phase
- Train finance staff — any wire transfer request over a threshold requires phone verification with known number, not callback to number in email
- Deploy email security gateway (Proofpoint, Mimecast) for additional filtering

**For SOC Teams:**
- Create alerts for Reply-To/From mismatches in email gateway logs
- Monitor for external emails spoofing internal domains
- Run regular phishing simulations to test staff awareness
- Review DMARC aggregate reports weekly

---

## Files in This Project

```
Project-3-Phishing-Analysis/
├── README.md                    ← This file
├── email-spoofing-explained.md  ← Deep dive on SPF, DKIM, DMARC
├── incident-report.md           ← Formal phishing incident report
└── screenshots/
    ├── 01-sendmail-install.png
    ├── 02-swaks-smtp-handshake.png
    ├── 03-email-headers-analysis.png
    └── 04-spf-dkim-dmarc-check.png
```

---

## Tools Used

| Tool | Purpose |
|---|---|
| **swaks** | SMTP testing and email spoofing simulation |
| **sendmail 8.18.2** | Local SMTP relay |
| **Kali Linux** | Attack simulation platform |
| **MXToolbox** | SPF/DKIM/DMARC record checking |
| **MITRE ATT&CK** | Threat classification |

---

*Analyst: Mwaura | SOC Portfolio Project 3 | April 2026*

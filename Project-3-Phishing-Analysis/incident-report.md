# Phishing Incident Report
## CEO Fraud / Wire Transfer Simulation

| Field | Details |
|---|---|
| **Report ID** | INC-2026-003 |
| **Date** | April 7, 2026 |
| **Analyst** | Mwaura |
| **Incident Type** | Phishing — CEO Fraud / Business Email Compromise |
| **Classification** | Simulated Lab Exercise |
| **Severity** | 🔴 HIGH (would be Critical in production) |
| **Status** | Documented |

---

## What Happened

I simulated a CEO fraud phishing attack where an attacker impersonates a company executive to trick the finance team into wiring money to an attacker-controlled bank account. This is one of the most financially damaging attack types in the real world.

The attack was carried out using swaks and sendmail on Kali Linux, with the spoofed sender set as `ceo@acmecorp.com` — a fictional company — and the target being my own email accounts (Gmail and Yahoo) for lab purposes only.

---

## Attack Details

**Spoofed Sender:** ceo@acmecorp.com
**Reply-To (Attacker):** attacker.finance@protonmail.com
**Target:** Finance team member (simulated)
**Requested Action:** $50,000 wire transfer
**Sending Infrastructure:** Kali Linux localhost via sendmail

**The social engineering angle used:**
- Impersonating the CEO (authority)
- Urgency — "before close of business today"
- Secrecy — "do not discuss with anyone"
- Specific bank details to make it feel legitimate

These three elements — authority, urgency, secrecy — are the standard formula for CEO fraud. The goal is to make the target act before they think.

---

## SMTP Handshake Evidence

The local sendmail server accepted the spoofed email without any authentication challenge:

```
=== Connected to 127.0.0.1
<- 220 kali ESMTP Sendmail 8.18.2/8.18.2/Debian-1
-> MAIL FROM:<ceo@acmecorp.com>
<- 250 2.1.0 <ceo@acmecorp.com>... Sender ok
-> RCPT TO:<mwaura090@gmail.com>
<- 250 2.1.5 <mwaura090@gmail.com>... Recipient ok
<- 250 2.0.0 Message accepted for delivery
```

This confirms that SMTP itself has no mechanism to verify the sender identity. The spoofed `MAIL FROM` was accepted without question.

---

## Email Authentication Results

| Check | Result | Detail |
|---|---|---|
| SPF | ❌ FAIL | acmecorp.com has no SPF record |
| DKIM | ❌ FAIL | No DKIM signature on the email |
| DMARC | ⚠️ NO POLICY | No DMARC record configured |
| Gmail Delivery | 🛡️ BLOCKED | Gmail's internal filters rejected silently |
| Yahoo Delivery | 🛡️ BLOCKED | Yahoo also rejected |

---

## What This Means in the Real World

Gmail and Yahoo blocked the email — but that's because they're among the most hardened consumer mail providers. The same attack against a target running their own mail server, or using a provider with weaker filtering, would likely succeed if:

- The target domain has no SPF record or uses `~all` instead of `-all`
- DMARC is set to `p=none` (monitoring only)
- The mail server is configured to accept all inbound mail regardless of authentication
- The target employee hasn't been trained to recognize CEO fraud patterns

The FBI reported over $2.9 billion in BEC losses in 2023. The attack vector in most of those cases was exactly this — spoofed or lookalike email, urgency, financial request.

---

## Indicators of Compromise

```
Attack Type      : CEO Fraud / Business Email Compromise
Spoofed Sender   : ceo@acmecorp.com
Real Reply-To    : attacker.finance@protonmail.com
Sending Server   : localhost (127.0.0.1) via sendmail
Tool Used        : swaks v20240103.0
Date/Time        : 2026-04-07 23:11:58 -0400
Message-ID       : 20260407231158.009335@kali
MITRE Technique  : T1566 — Phishing
```

---

## Detection Guidance

If I were monitoring email logs in a SOC environment, here's what I'd flag:

1. **Reply-To address differs from From address** — especially when Reply-To is a free webmail service like ProtonMail or Gmail on a corporate email
2. **Received header shows unexpected origin** — email claiming to be from acmecorp.com but arriving from a residential IP or unknown server
3. **SPF/DKIM failures on high-value domains** — finance, HR, executive addresses should be monitored closely
4. **Keywords in subject lines** — "urgent", "wire transfer", "confidential", "do not discuss" in combination
5. **X-Mailer anomalies** — legitimate email clients don't show swaks or Python scripts in the mailer header

---

## Response Actions (Simulated)

| Action | Status |
|---|---|
| Identified spoofed sender via header analysis | ✅ |
| Confirmed SPF/DKIM/DMARC failures | ✅ |
| Traced sending infrastructure to localhost | ✅ |
| Documented IOCs | ✅ |
| Verified email blocked by Gmail and Yahoo filters | ✅ |
| No funds transferred (simulation) | ✅ |

---

## Recommendations

**Immediate (for any organization):**
- Audit SPF records — switch from `~all` to `-all`
- Implement DKIM signing on all outbound mail
- Move DMARC from `p=none` to `p=quarantine` then `p=reject`
- Enable DMARC reporting and review weekly

**Process Controls:**
- Any wire transfer above a set threshold requires secondary verification — phone call to a known number, never callback to a number provided in the email
- Finance teams should be trained specifically on CEO fraud patterns — the authority/urgency/secrecy combination should be an automatic red flag
- Consider implementing a dual-approval policy for large transfers

**SOC Monitoring:**
- Create alerts for external emails with Reply-To mismatches
- Monitor for lookalike domains (acmec0rp.com, acmecorp-ltd.com)
- Run quarterly phishing simulations to test staff

---

## Lessons From This Lab

The most interesting takeaway from this exercise was that the attack technically succeeded at the SMTP level — sendmail accepted the spoofed email without hesitation. The defense came from Gmail and Yahoo's filtering, not from the protocol itself.

That gap — between what SMTP allows and what modern filters block — is exactly where attackers operate. Organizations relying on email provider filtering alone, without proper SPF/DKIM/DMARC configuration, are one misconfigured filter away from a successful BEC attack.

---

*Mwaura — SOC Portfolio Project 3 | April 2026*

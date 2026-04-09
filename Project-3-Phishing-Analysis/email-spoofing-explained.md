# 📨 Email Spoofing Explained — SPF, DKIM & DMARC Deep Dive

*My notes from researching and simulating email spoofing as part of Project 3*

---

## The Core Problem

Email was not built with security in mind. SMTP dates back to 1982 and the original designers never anticipated it would carry sensitive financial requests or be weaponized at scale. The `From:` field in an email is just a string — anyone can put anything there. That's it. That's the vulnerability.

Everything that came after — SPF, DKIM, DMARC — is essentially a retrofit. Bolting authentication onto a protocol that never had it.

---

## SPF — Sender Policy Framework

### What It Is
SPF is a DNS TXT record that lists which IP addresses or mail servers are authorized to send email on behalf of a domain.

Example SPF record for acmecorp.com:
```
v=spf1 include:_spf.google.com ip4:203.0.113.10 -all
```

This says: "Only Google's mail servers and IP 203.0.113.10 are allowed to send email from acmecorp.com. Reject everything else (`-all`)."

### How It Stops Spoofing
When Gmail receives an email claiming to be from `ceo@acmecorp.com`, it checks acmecorp.com's SPF record. If the sending IP isn't listed, SPF fails.

### The Weakness
SPF only checks the `MAIL FROM` (envelope sender) — not the `From:` header that users actually see. Attackers can pass SPF on the envelope while still spoofing the visible From header. This is called a **display name attack**.

Also, many domains still use `~all` (soft fail) instead of `-all` (hard fail), meaning SPF failures result in the email being marked suspicious rather than rejected outright.

### Checking SPF Records
```bash
# Check SPF record for any domain
dig TXT acmecorp.com | grep spf

# Or use nslookup
nslookup -type=TXT acmecorp.com
```

---

## DKIM — DomainKeys Identified Mail

### What It Is
DKIM adds a cryptographic signature to outgoing emails. The sending mail server signs the email with a private key, and the public key is published in DNS. Receiving servers verify the signature.

### What It Proves
- The email actually came from the domain's mail server
- The email wasn't modified in transit (integrity check)

### Example DKIM Header
```
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed;
  d=acmecorp.com; s=google;
  h=from:to:subject:date;
  bh=47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=;
  b=ABC123...signature...XYZ
```

### The Weakness
DKIM only works if the domain has set it up. Many domains — especially smaller organizations — skip it entirely because it requires DNS configuration. Without DKIM, there's nothing to verify.

---

## DMARC — Domain-based Message Authentication Reporting and Conformance

### What It Is
DMARC ties SPF and DKIM together and tells receiving servers what to do when checks fail. It also sends reports back to the domain owner so they can see who's sending on their behalf.

### DMARC Policy Options
```
p=none      → Do nothing, just send me reports (monitoring mode)
p=quarantine → Put failing emails in spam
p=reject    → Reject failing emails outright
```

### Example DMARC Record
```
v=DMARC1; p=reject; rua=mailto:dmarc@acmecorp.com; ruf=mailto:dmarc@acmecorp.com
```

### The Weakness
Many organizations set `p=none` and never move past monitoring mode. This means they get reports but spoofed emails still deliver. It's better than nothing but doesn't actually protect recipients.

---

## How My Spoofing Lab Exposed These Gaps

In my lab simulation, I spoofed `ceo@acmecorp.com` — a fictional domain with no SPF, DKIM or DMARC records. Here's what each check returned:

| Check | Result | Reason |
|---|---|---|
| SPF | FAIL | No SPF record for acmecorp.com |
| DKIM | FAIL | No DKIM signature present |
| DMARC | NO POLICY | No DMARC record for acmecorp.com |

Gmail rejected the email anyway based on its own internal reputation systems. But a less sophisticated mail server — or one configured to accept mail regardless of authentication failures — would have delivered it straight to the inbox.

---

## Checking Any Domain's Email Security Posture

These commands let you quickly assess how well any domain is protected:

```bash
# Check SPF
dig TXT targetdomain.com | grep spf

# Check DMARC
dig TXT _dmarc.targetdomain.com

# Check DKIM (need to know the selector — often 'google' or 'default')
dig TXT google._domainkey.targetdomain.com
```

Or use MXToolbox online: `mxtoolbox.com/SuperTool.aspx`

A domain with no SPF, no DKIM and no DMARC is essentially unprotected against spoofing. Attackers actively look for these gaps.

---

## Real World Impact

CEO fraud (Business Email Compromise) cost organizations over **$2.9 billion in 2023** according to the FBI IC3 report. The majority of those attacks used email spoofing or lookalike domains — not sophisticated malware, just manipulated email headers and social engineering.

The technical barrier to launching this attack is low. The swaks command I used in this lab takes about 30 seconds to write. The real defense is a combination of proper email authentication (SPF/DKIM/DMARC) and user awareness training.

---

*Mwaura — April 2026*

# Threat Intelligence Tools — Analysis Report

**Analyst:** Emma Gleeson
**Date:** August 2026
**Source material:** TryHackMe — Threat Intelligence Tools (SOC Level 1 Path)
**Tools covered:** UrlScan.io, Abuse.ch (ThreatFox, SSL Blacklist, URLHaus, FeodoTracker), PhishTool, Cisco Talos Intelligence
**Sample artifacts analysed:** Email1.eml, Email2.eml, Email3.eml

---

## 1. Objective

Build practical familiarity with core open-source threat intelligence
platforms used for IOC enrichment, and apply them against real phishing
email samples in a SOC-analyst-style triage scenario — moving from raw
indicator (a domain, IP, hash, or attachment) to an enriched, actionable
finding.

---

## 2. Methodology

Each tool was studied for its core function and IOC type, then applied
directly against provided evidence — either a reference scan
(TryHackMe's own domain) or a suspicious email sample requiring
investigation. Findings were cross-referenced across tools where
possible (e.g., an IP identified via email headers was then enriched
using Cisco Talos).

---

## 3. Threat Intelligence Classifications

Threat intelligence is geared toward understanding the relationship
between an organisation's operational environment and its adversaries.
It breaks down into four classifications:

| Classification | Focus |
|---|---|
| **Strategic** | High-level view of the organisation's threat landscape — trends, patterns, and emerging threats that inform business decisions |
| **Technical** | Evidence and artefacts of an attack (IOCs) — used by IR teams to baseline an attack surface and build defences |
| **Tactical** | Adversary tactics, techniques, and procedures (TTPs) — strengthens security controls via real-time investigation |
| **Operational** | Adversary motive and intent — helps security teams understand which critical assets (people, processes, technology) are likely targets |

**Applied read:** the tool-by-tool work in this report sits mainly at
the **Technical** level (IOCs like IPs, hashes, fingerprints) with a
layer of **Tactical** insight where indicators were attributed to a
specific malware family (Dridex) — moving from "here is an artefact"
to "here is the adversary behaviour/tooling behind it."

---

## 4. Tool Reference & Applied Findings

### 4.1 UrlScan.io
**What it does:** scans and catalogues a URL/domain, capturing DNS
resolution, hosting infrastructure, linked domains, and registrar
information without requiring the analyst to visit the site directly.

**IOC type:** URL / domain

**Applied example — tryhackme.com scan:**

| Field | Value |
|---|---|
| Cisco Umbrella Rank | 345612 |
| Domains identified | 13 |
| Domain registrar | NAMECHEAP INC |
| Main IP address | 2606:4700:10::ac43:1b0a |

**Why it matters:** gives an analyst a safe, immediate snapshot of a
domain's infrastructure and reputation signals before deciding whether
further investigation or blocking action is warranted.

---

### 4.2 Abuse.ch (ThreatFox, SSL Blacklist, URLHaus, FeodoTracker)
**What it does:** a suite of free trackers maintained by abuse.ch,
each focused on a different IOC type — malware C2 infrastructure
(ThreatFox), malicious TLS/SSL certificates (SSL Blacklist), malware
distribution URLs (URLHaus), and botnet C2 servers (FeodoTracker).

**IOC type:** IP:port, JA3 fingerprint, ASN, botnet IP

**Applied examples:**

| IOC | Tool | Finding |
|---|---|---|
| `212.192.246.30:5555` | ThreatFox | Malware alias: **Katana** |
| JA3 `51c64c77e60f3980eea90869b68c58a8` | SSL Blacklist | Associated malware: **Dridex** |
| ASN **AS14061** | URLHaus (stats) | Malware-hosting network: **DIGITALOCEAN-ASN** |
| Botnet IP `178.134.47.166` | FeodoTracker | Associated country: **Georgia** |

**Why it matters:** these trackers let an analyst quickly confirm
whether a raw indicator (an IP, a certificate fingerprint, a hosting
ASN) is already known and attributed to a specific malware family or
campaign — turning an unknown indicator into an actionable, labelled
threat.

---

### 4.3 PhishTool
**What it does:** a dedicated phishing-email analysis platform for
parsing email headers, sender infrastructure, and message content in a
structured way — purpose-built for the exact triage workflow a SOC
analyst runs on a reported phishing email.

**IOC type:** email headers, sender IP, sender domain

**Applied example — Email1.eml:**

| Field | Value |
|---|---|
| Platform impersonated | LinkedIn |
| Sender address | darkabutla@sc500.whpservers.com |
| Recipient address | cabbagecare@hotsmail.com |
| Originating IP (defanged) | 204[.]93[.]183[.]11 |
| Mail hops to recipient | 4 |

**Why it matters:** structured header parsing surfaces exactly the
fields (originating IP, hop count, sender infrastructure) needed to
assess authenticity and trace an email's true origin — the same fields
manually inspected in earlier phishing analysis work in this
portfolio, but extracted here via a dedicated tool rather than manual
header reading.

---

### 4.4 Cisco Talos Intelligence
**What it does:** provides IP/domain reputation and ownership lookup,
including network owner (WHOIS-style) and customer/organisation
attribution — used here to enrich the IP identified from Email1.eml.
Notably, the analysis VM from the PhishTool task had no internet
access, so this enrichment step required a separate, connected lookup
— a realistic constraint mirroring isolated malware-analysis
environments in real SOC settings.

**IOC type:** IP address

**Applied example — enriching the Email1.eml originating IP:**

| Field | Value |
|---|---|
| Network owner | DEFT.COM |
| Customer name | Complete Web Reviews |

**Why it matters:** demonstrates the natural next step in an
enrichment workflow — an indicator extracted from one tool (PhishTool)
is fed into a second tool (Talos) to add ownership/attribution context,
rather than treating each tool as a standalone lookup.

---

## 5. Applied Scenarios — Email Triage

### 5.1 Scenario 1 — Email2.eml
**Task:** triage a coworker-forwarded suspicious email using tools
covered in this room.

| Field | Value |
|---|---|
| Recipient address | chris.lyons@supercarcenterdetroit.com |
| VirusTotal — first submission year (attached file) | 2017 |

**Finding:** the attached file's presence on VirusTotal since 2017
indicates a long-known, previously catalogued sample rather than a
novel threat — useful context for prioritisation, since an established
IOC is faster to confirm as malicious than an unseen one.

### 5.2 Scenario 2 — Email3.eml
**Task:** triage a second forwarded suspicious email.

| Field | Value |
|---|---|
| Attachment name | Sales_Receipt 5606.xls |
| Associated malware family | Dridex |

**Finding:** notably, this attachment is attributed to **Dridex** —
the same malware family identified earlier via the SSL Blacklist JA3
fingerprint lookup (Section 4.2). This cross-reference suggests a
consistent campaign or actor using shared infrastructure/tooling
across multiple observed indicators, rather than unrelated incidents.

---

## 6. Cross-Tool Correlation

| Indicator | Source | Attribution |
|---|---|---|
| JA3 `51c64c77e60f3980eea90869b68c58a8` | SSL Blacklist | Dridex |
| `Sales_Receipt 5606.xls` (Email3.eml attachment) | VirusTotal / room analysis | Dridex |

**Applied read:** two independently investigated indicators — a TLS
fingerprint and a malicious email attachment — both trace back to the
same malware family. In a real SOC context, this kind of correlation
across separate alerts is exactly what justifies escalating from
"isolated suspicious email" to "part of a broader Dridex campaign
targeting the organisation," changing both the priority and the scope
of the response.

---

## 7. Findings Summary

| Task | Finding |
|---|---|
| UrlScan.io (tryhackme.com) | Cisco Umbrella Rank 345612; 13 domains; registrar NAMECHEAP INC; IP 2606:4700:10::ac43:1b0a |
| ThreatFox | `212.192.246.30:5555` → Katana |
| SSL Blacklist | JA3 `51c64c77...` → Dridex |
| URLHaus | AS14061 → DIGITALOCEAN-ASN |
| FeodoTracker | `178.134.47.166` → Georgia |
| PhishTool (Email1.eml) | LinkedIn impersonation; originating IP `204[.]93[.]183[.]11`; 4 hops |
| Cisco Talos (IP enrichment) | Owner: DEFT.COM; Customer: Complete Web Reviews |
| Scenario 1 (Email2.eml) | VirusTotal first seen: 2017 |
| Scenario 2 (Email3.eml) | Attachment `Sales_Receipt 5606.xls` → Dridex |

---

## 8. Conclusion

This room demonstrated the full IOC enrichment lifecycle: extracting a
raw indicator from a suspicious artifact (an email, a certificate, an
attachment), enriching it through a purpose-built tool, and — critically
— correlating findings *across* tools and *across* separate incidents
to identify a shared campaign. The Dridex correlation between the SSL
Blacklist lookup and the Email3.eml attachment is the clearest example
of this: no single tool would have surfaced that connection alone, but
combining findings across two workflows did. Framed against the four
classifications in Section 3, this work sits primarily at the
Technical level (raw IOCs) with a clear step up into Tactical insight
once those IOCs were attributed to adversary tooling — the same
correlation habit, applied consistently, is what moves an analyst from
reactive alert-closing toward genuine threat intelligence.

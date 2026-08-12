# Phishing Response Playbook

**Purpose:** Standard operating procedure for triaging and responding
to a reported or detected phishing email — generalised from patterns
observed across six real phishing case studies (see
`Writeups/Phishing-analysis-01.md`).

**Scope:** Applies to phishing delivered via link, tracking pixel,
multi-stage redirect, or malicious attachment. Does not cover active
malware execution response (see separate Malware Alert Triage
Playbook once created).

---

## 1. Detection

Phishing typically enters the triage queue via one of:
- User-reported email (forwarded or flagged via a report-phishing button)
- Email security gateway alert (e.g., Proofpoint)
- SIEM correlation rule (e.g., unusual volume of similarly-structured
  emails, known-bad sender domain)

**On receipt, capture immediately:**
- Full raw email / `.eml` file (never rely on a screenshot alone)
- Reporting user and timestamp
- Any user-reported action already taken (e.g., "I clicked the link")

---

## 2. Triage — Header & Sender Analysis

**Check first (fastest, highest-yield checks):**

| Check | What to look for |
|---|---|
| Display name vs. actual sending address | Mismatch is the single most consistent red flag across observed cases |
| Sender domain | Compare against the organisation's real domain — lookalike/unrelated domains are common |
| BCC delivery | Recipient BCC'd rather than directly addressed suggests a bulk/mass campaign, not targeted spear-phishing |
| Urgency language in subject/body | Same-day deadlines, "account suspended," "action required" — a deliberate pressure tactic |

**Escalate priority immediately if:**
- Sender spoofs an internal/executive address (possible BEC)
- Recipient is a privileged/admin account
- Multiple reports reference the same sender/subject in a short window

---

## 3. Investigation

**Step through in this order — stop and escalate if a step confirms malicious intent:**

1. **Inspect links via raw HTML source**, not the rendered button/text.
   Visible link text is not reliable — always check the actual `href`.
2. **Check for URL shortening/obfuscation.** If present, resolve the
   destination using a safe third-party unshortening tool (never visit
   directly).
3. **Check for tracking pixels** (0-pixel embedded images). Presence
   alone confirms the sender can detect that the mailbox is active,
   independent of any link click — relevant even if no link was
   clicked.
4. **Trace multi-stage redirects fully**, not just the first hop.
   Brand-impersonation chains often layer multiple trusted brands
   (e.g., Citrix → OneDrive → Adobe) specifically to compound
   perceived legitimacy before a credential prompt.
5. **If a login/credential form is reached:** do not enter real
   credentials. If already tested in a sandboxed context, note whether
   the form returns a generic error regardless of input — this
   confirms the form is not performing real authentication and exists
   solely to harvest input.
6. **If an attachment is present:**
   - Note the file type — unusual formats (e.g., `.dot` template files)
     are themselves a signal
   - Open only in an isolated/sandboxed environment
   - Check for embedded links, macros, or "enable editing/content"
     prompts (a common trigger for macro-based payloads)
   - Check for internal content inconsistencies (e.g., conflicting
     sender country, invoice address, and document language) — a
     reliable signal independent of header analysis
7. **If an executable or script is referenced or triggered:** do not
   run it outside an isolated environment. Note the intended
   filename/payload and treat as a malware delivery case, not just
   credential harvesting.

---

## 4. Containment

**Standard actions once malicious intent is confirmed:**
- Block sending domain and originating infrastructure at the email gateway
- Search mailbox/mail logs for other recipients of the same
  sender/subject pattern
- If any user reached a credential-harvesting page: force a password
  reset for that user, regardless of whether they believe they entered
  real credentials
- If a malicious attachment was opened: isolate the affected endpoint
  pending further investigation (do not assume the payload failed to
  execute)
- Submit confirmed malicious domains/URLs/hashes to the organisation's
  threat intelligence platform (e.g., MISP) for future detection

---

## 5. Documentation

**Every case should record, at minimum:**
- Sender (display name + actual address)
- Subject line
- Delivery method (link / attachment / tracking pixel / multi-stage)
- MITRE ATT&CK mapping (T1566.001 for attachment-based, T1566.002 for
  link-based)
- Techniques identified (spoofing, urgency, brand impersonation, etc.)
- Recommendation and containment actions taken
- Whether the case correlates with any other recent phishing activity
  (shared infrastructure, malware family, campaign pattern)

**Escalation note:** if a case's attachment or infrastructure matches
a known malware family or a previously seen IOC (e.g., via threat
intel enrichment tools), flag it explicitly as part of a broader
campaign rather than an isolated incident — this changes response
priority.

---

## Quick Reference — Red Flags by Frequency

Based on six observed cases, ranked by how consistently each appeared:

1. Display name / actual address mismatch — **6/6 cases**
2. Urgency framing — **4/6 cases**
3. BCC bulk delivery — **2/6 cases**
4. Blank body / attachment-only delivery (evades body-text filtering) — **2/6 cases**
5. Conflicting internal document details (geography, language) — **1/6 cases**, but high-signal when present

**Takeaway:** the display-name/address check costs seconds and would
flag the majority of phishing attempts before any deeper investigation
is needed — always perform this check first.

# Phishing Emails in Action — Case Studies (TryHackMe)

## Overview
Analysis of four real-world phishing samples covering distinct delivery
and evasion techniques: spoofed-brand receipts, tracking-pixel-laden
shipping notifications, multi-stage credential-harvesting redirect
chains, and attachment-based brand impersonation. Methodology followed
safe-handling practice throughout — no live IOC (domain, IP, or
attachment) was directly interacted with; URL expansion was done via
a third-party unshortening tool (WhereGoes) rather than direct navigation.

---

## Case 1: Spoofed PayPal Receipt ("Cancel Your Order")

**What I saw**
- Subject: "Your Receipt for Payment to Amazing Stuff"
- Display sender: service@paypal.com
- Actual sender: noreplyexmhwje9hhzdglnve9ml1xqzypnedg51ny7yleqejbwe@sultanbogor.com
- Recipient address: an atypical, non-standard Yahoo-domain address
- Body: branded HTML mimicking a genuine PayPal gift-card purchase receipt

**What I suspected**
Display name/actual address mismatch was the immediate tell — no
legitimate PayPal email would send from an unrelated domain like this.

**What I checked**
- Compared display name vs. raw sending address — confirmed mismatch
- Inspected the "Cancel the order" button's underlying HTML (raw source)
  rather than trusting the visible text
- Found the href pointed to a shortened URL: `https://is.gd/6oCJ4m`
- Used WhereGoes (safe unshortening tool) to resolve the destination
  without visiting it directly — resolved to an unrelated
  Google consent/redirect page, confirming the link had no legitimate
  connection to PayPal

**What I concluded**
Spoofed-brand phishing using URL shortening specifically to defeat
"hover to preview" checks — the visible link text and button branding
gave no indication of the real (unrelated, suspicious) destination.

**Techniques identified:** spoofed email address, URL shortening/obfuscation, branded HTML impersonation

**MITRE ATT&CK:** T1566.002 – Spearphishing Link

---

## Case 2: Shipping Notification with Tracking Pixel ("Track Your Package")

**What I saw**
- Subject: "Track your package: # LZ8942357486EN"
- Display sender: "Distribution Center"
- Actual sender: contact@beginpro.club
- Yahoo automatically blocked images/links in this email

**What I suspected**
Display-name spoofing again, but Yahoo's automatic image-blocking
suggested something beyond a simple bad link — likely a tracking
mechanism.

**What I checked**
- Reviewed the raw HTML source rather than relying on hover-preview
  (Yahoo had disabled link previews entirely)
- Found the visible "Track your package" link resolved to
  `http://devret.xyz/4833mt...` — unrelated to any shipping carrier
- Identified an embedded 0-pixel image tag (`Tracking.png`) — a
  classic tracking pixel used to notify the attacker's server the
  moment the email is opened, regardless of whether any link is clicked
- Noted the recipient's email address was embedded directly in one of
  the tracking image URLs — meaning simply opening the email (with
  images enabled) would confirm to the attacker that this specific
  address is active and monitored

**What I concluded**
This sample's real payload isn't just the malicious link — it's
reconnaissance. The tracking pixel alone, independent of any click,
confirms a live, monitored inbox to the attacker, which typically
leads to further targeted follow-up phishing.

**Techniques identified:** spoofed sender, pixel tracking, link manipulation

**MITRE ATT&CK:** T1566.002 – Spearphishing Link; reconnaissance value comparable to T1589 (Gather Victim Identity Information) from the attacker's side

---

## Case 3: Multi-Stage Credential Harvesting ("Download Document Here")

**What I saw**
- From: Nir Barak <nirb@profility.com>
- Subject referenced a fake insurance claim/realty case number
- Sent Thu 7/15/2021 1:35 PM — document link stated to expire
  "July 15, 2021" — the **same day**, creating an artificially narrow
  action window
- Body impersonated a Citrix secure-file-share notification with a
  "Download Document Here" button

**What I suspected**
The same-day expiry was a strong urgency-manipulation signal before
even reaching the link itself.

**What I checked**
- Traced the full redirection chain rather than stopping at the first hop:
  1. "Download Document Here" → landing page impersonating **OneDrive**
     (URL: app.popt.in/landing/... — clearly not a Microsoft domain)
  2. Clicking through → second redirect impersonating **Adobe Document
     Cloud** (URL: bdkmotorsport.com/wp-duu.a/ — unrelated business domain)
  3. Final stage: a fake login prompt asking the victim to
     authenticate via Outlook, Office365, or "Other Mail" to "view
     the document"
- Tested the login form logic (in an isolated/sandboxed context) —
  entering any credentials returned a generic "Invalid Credentials"
  error regardless of input, confirming the form does **not** perform
  real authentication — it exists solely to capture and forward
  whatever is typed to the attacker

**What I concluded**
A three-brand impersonation chain (Citrix → OneDrive → Adobe) designed
to exploit cumulative trust — each stage borrows legitimacy from a
different well-known service, making the victim less likely to
question the final credential prompt. The fake "Invalid Credentials"
error is a deliberate design choice to mask the harvest and encourage
repeated (multiple) credential entry attempts.

**Techniques identified:** artificial urgency, layered brand impersonation, multi-stage link redirection, credential harvesting via fake login portal

**MITRE ATT&CK:** T1566.002 – Spearphishing Link; T1589 relevant on the attacker's reconnaissance side once credentials are captured

---

## Case 4: Attachment-Based Brand Impersonation ("Your Account Is on Hold")

**What I saw**
- Subject: "Netlfix ID Suspended" — note the typo (Netlfix, not Netflix)
- Display sender: "Netflix billing"
- Actual sender: z99@musacombi.online
- Recipient was **BCC'd** — a strong indicator of a bulk/mass phishing
  campaign rather than a targeted send
- Body used Netflix branding/logo via rendered HTML, stated the
  account was "on hold" pending a billing update
- Payload delivered via a **PDF attachment**, not a direct link

**What I suspected**
The subject-line typo and BCC delivery suggested a lower-effort,
high-volume campaign compared to Case 3's more deliberate chain.

**What I checked**
- Confirmed display name/domain mismatch (billing claim vs.
  unrelated .online domain)
- Reviewed the PDF attachment's embedded content rather than opening
  it directly — found an embedded "Update Payment Account" link
  pointing to a URL with no connection to Netflix's actual domain
- Noted a listed support phone number in an atypical international
  format, inconsistent with Netflix's genuine contact channels
- Noted the email also referenced a **legitimate**-looking
  "help.netflix.com" style link elsewhere in the body — a deliberate
  mix of real and fake references to boost perceived credibility

**What I concluded**
Using a PDF attachment (rather than an inline link) is a specific
evasion choice — many email security filters weight inline links more
heavily than links embedded inside an attached document, so this
routes the same credential-harvesting intent around lighter-weight
filtering.

**Techniques identified:** spoofed display name, urgency, brand impersonation, poor grammar/typos, attachment-based link concealment

**MITRE ATT&CK:** T1566.001 – Spearphishing Attachment

---

## Case 5: Blank-Body Attachment Phishing ("Your Recent Purchase")

**What I saw**
- Subject: "Re: Action Required - Your recent purchase 'Double Jackpot
  Slots Las Vegas' on the App Store"
- Display sender: "Apple Support"
- Actual sender: donoreply-storemailsedmopzl07zo7o9@sumpremed.com
- To: a malformed/typo'd address (noreply.payament_@app.apple.iud456.com)
- Recipient was **BCC'd** (same bulk-campaign pattern as Case 4)
- Email body was **completely blank** — the only content was a `.dot`
  (Microsoft Word Template) attachment

**What I suspected**
A blank body paired with an attachment is itself a red flag — legitimate
purchase receipts always contain body content; an empty email whose
entire purpose is "open this file" is a strong sign the file is the
actual attack vector.

**What I checked**
- Confirmed display name/domain mismatch, plus visible typos in both
  the From and To addresses
- Noted the `.dot` file format itself as unusual — a Word *template*
  file is an atypical, and slightly less scrutinized, format for
  what's meant to be a receipt
- Opened the document in an isolated context and found a large
  embedded image (styled as an Apple purchase receipt) acting as a
  clickable redirect rather than static content
- Traced the embedded redirect: `t.tumblr.com/redirect?z=...` chaining
  to `apps.ios-games.mansiiea.com` — the excessive URL length and
  unrelated final domain (despite containing legitimate-sounding terms
  like "apps" and "ios") confirmed it as a phishing redirect rather
  than a genuine Apple/App Store domain

**What I concluded**
The entire "phishing surface" here is the attachment, not the email
body — a deliberate choice likely intended to slip past body-text-based
spam filtering, since there's no suspicious language in the message
itself to flag.

**Techniques identified:** spoofed sender, BCC bulk delivery, blank-body/attachment-only delivery, unusual file format (.dot), long/obfuscated redirect URL

**MITRE ATT&CK:** T1566.001 – Spearphishing Attachment

---

## Case 6: Malicious Excel Attachment with Executable Payload ("Scheduled Shipment")

**What I saw**
- Subject: "DHL Express Courier Shipping notice CBJ200620039539"
- Display sender: "DHL Express"
- Actual sender: info@glamcarcompany.de — an unrelated German domain
  with no connection to DHL
- Body used genuine-looking DHL branding/HTML, but had minimal text
  content — the real payload was in the attached `.xlsx` file

**What I suspected**
Given the pattern from Case 5, an attachment-heavy, low-body-text
email suggested the actual attack lived in the file, not the message.

**What I checked**
- Opened the `.xlsx` attachment in an isolated context and immediately
  noticed **conflicting geographic/language markers**: the sender
  domain was German, the invoice was addressed to a company in India,
  but the spreadsheet's actual content was written in **Mandarin** —
  three unrelated regions in one document, a strong internal
  inconsistency red flag
- Found a single embedded hyperlink styled as "click enable editing"
  — a common social-engineering trick used specifically to get victims
  to enable macros/active content in Office documents
- Traced what happens when the link/macro is triggered: it attempts
  to download and execute a file named `regasms.exe`
- Observed the execution attempt directly — it produced a system
  compatibility error in the sandboxed environment, but the attempted
  execution itself confirmed malicious intent (the error prevented
  the payload from running here, not the attacker's code from trying)

**What I concluded**
This sample escalates well beyond credential harvesting — it's a
direct **malware delivery mechanism** disguised as a shipping notice.
Had the executable run successfully, potential outcomes include
persistence (backdoor/scheduled task), data/credential exfiltration,
or ransomware deployment.

**Techniques identified:** spoofed sender/brand impersonation, conflicting geographic/language markers as a detection signal, macro-enablement social engineering, embedded executable payload

**MITRE ATT&CK:** T1566.001 – Spearphishing Attachment; T1204.002 – User Execution: Malicious File; T1059 – Command and Scripting Interpreter (relevant once the `.exe` would execute)

---

## Patterns Observed Across All Six Samples

1. **Display name ≠ actual sending address** — present in every single
   case (6/6) — the most consistent, checkable red flag across the
   entire sample set
2. **Urgency framing** appeared in 4 of 6 samples — a deliberate
   pressure tactic to short-circuit careful review
3. **BCC delivery** appeared in Cases 4 and 5 — a reliable signal of
   bulk/mass campaigns rather than targeted spear-phishing
4. **Attack surface shifted from body text to attachments** in the
   later cases (5 and 6) — both used blank or minimal body content,
   suggesting a deliberate attempt to evade body-text-based spam
   filtering by pushing all malicious content into the file itself
5. **Severity escalated across the sample set** — from simple
   credential harvesting (Cases 1–4) to direct executable malware
   delivery (Case 6) — showing phishing isn't a single fixed outcome,
   but a delivery mechanism for a range of attacker goals
6. **Conflicting internal details** (mismatched geography/language in
   Case 6, nonsensical wording in Case 3) are a detection signal
   independent of domain/header analysis — worth checking document
   *content* consistency, not just metadata

   ---

   | Case | Sample | Technique |
|---|---|---|
| 1 | PayPal receipt | T1566.002 – Spearphishing Link |
| 2 | Package tracking | T1566.002 – Spearphishing Link |
| 3 | Document download | T1566.002 – Spearphishing Link |
| 4 | Netflix billing | T1566.001 – Spearphishing Attachment |
| 5 | Apple purchase | T1566.001 – Spearphishing Attachment |
| 6 | DHL shipment | T1566.001 – Spearphishing Attachment; T1204.002 – User Execution: Malicious File |

---

## Conclusion
Across six samples, this room demonstrated that phishing is not a
single technique but a spectrum of delivery methods (direct link,
tracking pixel, multi-stage redirect, attachment) escalating toward a
range of attacker objectives (credential harvesting through to direct
malware execution). The single most reliable, checkable indicator
across every sample was the display-name-versus-actual-address
mismatch — a check that costs seconds and would have flagged all six
emails before any deeper investigation was needed. Later-stage
detection (redirect chains, document content inconsistencies, embedded
executables) required progressively more investigative effort,
reinforcing that layered defense — not any single check — is what
actually catches sophisticated phishing.

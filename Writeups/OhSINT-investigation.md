# OSINT Investigation — OhSINT (TryHackMe)

**Analyst:** Emma Gleeson
**Source material:** TryHackMe — OhSINT
**Starting point:** A single image file, `WindowsXP_1551719014755.jpg`
**Objective:** To determine what can be uncovered about an individual starting from nothing but one basic image file, using only open-source/publicly available information.

---

## Methodology

This investigation followed a linked pivot chain — each finding justified
the next step, rather than searching broadly at each stage. All checks
were passive (no direct interaction with the target's accounts or
infrastructure).

---

## Step 1 — Image Metadata Extraction

**Tool used:** A free web-based EXIF viewer (chosen over command-line
exiftool for speed, given AttackBox file-transfer issues encountered
mid-investigation — noted here as a real troubleshooting detour, not
just a clean first attempt).

**Finding:** The image's **Copyright** Metadata field contained a
username: `OWoodflint`

**Why this matters:** Copyright/Author fields are frequently
overlooked by people editing or exporting images — many editing tools
auto-populate this field with a saved username, making it a
consistently useful pivot point in image-based OSINT investigations.

---

## Step 2 — Username → Social Profile

**Method:** Direct search engine query for `OWoodflint`

**Finding:** Located an active X (Twitter) account under the handle
**@OWoodflint**, including a visible avatar and profile information.

---

## Step 3 — Social Profile → GitHub

**Finding:** The X profile linked directly to a GitHub account,
`OWoodfl1nt`, hosting a public repository (`people_finder`).

**Information gathered from the GitHub README:**

| Field | Value |
|---|---|
| Stated location | London |
| Personal email | OWoodflint@gmail.com |
| Personal blog | oliverwoodflint.wordpress.com |
| Confirmed X handle | @OWoodflint (self-linked, closing the loop with Step 2) |

**Why this matters:** Repository README "about me" sections are a
frequently under-guarded source of PII — written casually by
developers who don't consider them public intelligence in the same way
they might a social media bio.

---

## Step 4 — Social Media → WiFi Access Point Disclosure

**Finding:** An X post from March 2019 disclosed the BSSID of the
target's home wireless access point directly:

> "From my house I can get free wifi ;D
> Bssid: B4:5D:50:AA:86:41 - Go nuts!"

**Why this matters:** A BSSID is the fixed hardware (MAC) address of a
wireless access point. Unlike an IP address, it does not change
unless the physical device is relocated — making it a far more
durable and precise geolocation indicator than most information
people consider "safe" to post publicly.

---

## Step 5 — BSSID → Physical Location (WiGLE)

**Tool used:** wigle.net — a crowdsourced global database mapping
BSSIDs to real-world GPS coordinates, built from wireless network
scanning contributed by users worldwide.

**Finding:**

| Field | Value |
|---|---|
| SSID | `UnileverWiFi` |
| Approximate location | London (consistent with Step 3's stated location) |

**Why this matters:** This is the centerpoint finding of the
investigation — a single social media post made as a lighthearted
joke ("go nuts!") directly enabled cross-referencing a specific,
real-world physical location, corroborating and sharpening the
general "London" location already inferred from the GitHub README.

---

## Step 6 — Password Discovery via Source Code Inspection

**Method:** Having exhausted the primary visible resources (image
metadata, X, GitHub, WiGLE), the personal WordPress blog identified in
Step 3 was inspected directly — specifically, its **page source
code**, rather than just the rendered page content.

**Finding:** The password `pennYDr0pper.!` was embedded in the site's
source code — not visible anywhere in the normal rendered page.

**Why this matters:** This is a distinct technique from every prior
step in this chain — rather than following a visible link or reading
displayed content, this required manually inspecting the underlying
HTML/source, since sensitive information is sometimes left in code
comments, hidden fields, or unused markup that never renders visibly
to a normal visitor. A methodical OSINT investigation should always
include checking source code as a step, not just what's displayed on
screen.

---

## Findings Summary

| Step | Source | Finding |
|---|---|---|
| 1 | Image EXIF metadata | Username: `OWoodflint` |
| 2 | X (Twitter) | Active profile located |
| 3 | GitHub README | Location (London), email, blog link |
| 4 | X post | WiFi BSSID disclosed |
| 5 | WiGLE.net | SSID: `UnileverWiFi`; location corroborated |
| 6 | WordPress source code | Password: `pennYDr0pper.!` |

---

## Conclusion

This investigation demonstrates the core principle of OSINT: no single
piece of information gathered here was individually dangerous — a
username, a stated city, a WiFi BSSID shared as a joke, and a
password left in unrendered source code. Correlated together, they
produced a specific real-world identity, a precise physical location,
and valid credentials, starting from nothing but a single image file.

The methodology also highlighted two practical lessons worth carrying
into professional CTI/OSINT work:
1. **Passive social engineering artifacts** (a casually shared BSSID,
   an "about me" README) are frequently richer intelligence sources
   than deliberately protected ones — attackers and investigators
   alike exploit the gap between what people think is private and
   what is actually exposed.
2. **Source code inspection should be a standard step**, not an
   afterthought — the final and most sensitive finding in this
   investigation (the password) was invisible on the rendered page
   entirely, and would have been missed by relying on visible content
   alone.

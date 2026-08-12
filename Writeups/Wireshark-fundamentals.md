# Wireshark Fundamentals — Analysis Report

**Analyst:** Emma Gleeson
**Date:** August 2026
**Source material:** TryHackMe — Wireshark: The Basics
**Capture file analysed:** Exercise.pcapng
**Tool:** Wireshark

---

## 1. Objective

Build foundational proficiency in Wireshark's interface, protocol
dissection, packet navigation, and filtering capabilities, and apply
that proficiency against a real capture file to extract embedded
files, reconstruct HTTP conversations, and locate specific evidence —
practising the same core workflow used in packet-level SOC
investigations.

---

## 2. Methodology

Concepts were studied as presented in the source material, then
applied hands-on against `Exercise.pcapng` to extract artifacts,
resolve ambiguous or malformed data, and characterise the capture as a
whole using Wireshark's built-in statistics tooling. Each section below
pairs the concept with how it was actually used during analysis.

---

## 3. Tool Overview

| Feature | Description |
|---|---|
| Use cases | Live network monitoring (real-time capture on an interface) and forensic analysis (reviewing a previously recorded `.pcap`/`.pcapng` file) |
| GUI and data | Three-pane interface — packet list, packet details (expandable protocol tree), packet bytes (raw hex/ASCII) |
| Loading PCAP files | File → Open, to review previously recorded traffic |
| Colouring packets | Automatic colour-coding by protocol/state in the packet list (e.g., HTTP, TCP errors) for at-a-glance triage |
| Traffic sniffing | Live capture on a selected network interface — distinct from viewing a saved file (see Section 7) |
| Merging PCAP files | Combining multiple capture files for analysis spanning more than one recording session |

**Applied:** every subsequent task in this exercise relied on correctly
navigating the three-pane structure — this was the foundation for all
extraction and search work that followed.

---

## 4. Packet Dissection

Each packet is broken down layer by layer in the Packet Details pane,
mirroring the OSI model: **Frame → Ethernet II → Internet Protocol →
Transmission Control Protocol → Hypertext Transfer Protocol**, with
recognised file formats (e.g., JPEG File Interchange Format) appearing
as their own nested dissector layer.

**Applied — Packet 39765:** dissection traced the full chain from Frame
down to a nested JPEG object. The image had arrived via HTTP chunked
transfer encoding, meaning the raw single-packet bytes were not a
complete, valid file on their own. Wireshark's de-chunked entity body
reassembly (via the Show Packet Bytes dialog) was required to obtain
the clean, complete image prior to extraction and hashing.

**Supporting concepts:**
- **TTL (Time to Live):** hop-limit field preventing infinite packet
  loops; relevant to routing/path context at the IP layer
- **TCP payload:** the actual application data (the JPEG bytes) carried
  inside the TCP segment, distinct from header metadata (ports,
  sequence numbers, flags)

---

## 5. Packet Navigation

**Export Objects:** File → Export Objects → HTTP lists every file
Wireshark recognises as transferred across the capture — filename,
size, content type — without manual packet-by-packet searching.

**Applied:** used to locate a `.txt` file (`note.txt`) among several
`.txt` objects present in the capture, narrowed down by filename and
size before opening each candidate.

**Export Packet Bytes vs. Show Packet Bytes:**

| Method | Use case |
|---|---|
| Export Packet Bytes | Right-click a specific dissected field (e.g., "JPEG File Interchange Format") to save just that reassembled content |
| Show Packet Bytes (Ctrl+Shift+O) | Dedicated dialog with Frame / Reassembled TCP / De-chunked entity body tabs — required when data spans multiple TCP segments |

**Applied:** an initial raw single-packet export did not reliably
produce a valid JPEG, confirming the file had arrived chunked.
Right-clicking the dissected "JPEG File Interchange Format" layer
(rather than raw Frame bytes) correctly extracted the full 3,324-byte
reassembled image, which was then hashed successfully with `md5sum`.

**Follow HTTP/TCP Stream:** reconstructs an entire conversation as
continuous, readable text/HTML — required when the packet details view
truncates longer content.

**Applied:** an initial attempt followed the wrong stream (a homepage
GET request, then an unrelated image request). Re-running a targeted
search ("r4w", Packet details scope) confirmed the correct packet
(37061) before right-clicking → Follow → HTTP Stream. The correct
stream (`tcp.stream eq 12`) contained the artist listing HTML, where
the searched string `r4w8173` appeared as an artist's display name
inside an `<h3>` tag rather than as anchor text — a reminder to read
the actual markup structure rather than assume a pattern.

**Packet comments:** right-click → Packet Comment revealed
analyst-added notes attached to a specific packet, used here to
retrieve task instructions embedded directly in the capture file.

---

## 6. Packet Filtering

| Feature | Description |
|---|---|
| Apply as Filter / Prepare as Filter | Right-click any field to generate a display filter matching that value — Apply runs it immediately, Prepare loads it for editing first |
| Apply as Column | Turns any packet-details field into its own packet-list column |
| Conversation Filter | Isolates every packet belonging to a single conversation in one click |
| Colourise Conversation | Highlights a specific conversation's packets in the list with a distinct colour |
| Follow Stream | Reconstructs an entire TCP/HTTP conversation as continuous text (see Section 5) |
| Display filter queries | Hand-written filters (e.g., `tcp.stream eq 12`, `http`, `ip.addr==x.x.x.x`) typed directly into the filter bar |

**Applied — find bar (Ctrl+F):** used with scope set to **Packet
details** to search for strings inside dissected/parsed fields — e.g.,
searching "r4w" to locate the packet referencing a specific artist ID.

**Applied — display filter bar:** `tcp.stream eq 12` isolated every
packet belonging to the correct HTTP conversation once identified,
cross-checked against the Info column before following the stream.

---

## 7. Capturing vs. Viewing

**Capturing** is live interception of traffic on a network interface in
real time. **Viewing** is examining packets already recorded — either
live during an active capture, or from a previously saved `.pcapng`
file. This exercise was viewing throughout: `Exercise.pcapng` was
pre-recorded, with no live capture involved.

---

## 8. Additional Analysis (Self-Directed)

Beyond the assigned tasks, two further statistics views were used to
characterise the capture as a whole.

### 8.1 Protocol Hierarchy

- 58,620 total frames, almost entirely IPv4 (58,613 packets, 99.99%)
- 99.8% of packets were TCP (98.2% of total bytes) — confirming a
  single-protocol (TCP/HTTP) dataset rather than mixed traffic
- HTTP: 1.9% of packets but 6.1% of bytes — a higher byte-share than
  packet-share, consistent with HTTP carrying larger payloads per
  packet than surrounding TCP control traffic
- 9 JPEG File Interchange Format objects detected, 2 flagged as
  Malformed Packets — consistent with the chunked-transfer reassembly
  issue encountered when extracting the image from packet 39765,
  confirming this was a small but consistent pattern rather than an
  isolated case
- Minimal DNS (0.2%) and negligible TLS (0.1%) — consistent with a
  controlled, largely unencrypted HTTP dataset appropriate for a
  training exercise

### 8.2 Conversations (TCP)

- 19 distinct TCP conversations identified
- Two conversations dominate by volume:
  - `10.100.1.33:48924 → 10.10.57.178:80` — 28,083 packets, 54 MB
  - `10.100.1.33:43514 → 10.10.57.178:80` — 30,111 packets, 55 MB
- Every other conversation is under 32 KB — these two conversations
  account for the overwhelming majority of the capture's size and
  packet count

**Applied read:** sorting conversations by byte count is often the
fastest way to identify the "main event" in a capture. Here, the two
large `10.100.1.33 ↔ 10.10.57.178` conversations are clearly the
dataset's primary activity, while the many small conversations to
external IPs on ports 80/443/9696 read as routine background traffic
rather than requiring individual investigation.

---

## 9. Findings Summary

| Task | Finding |
|---|---|
| Extracted JPEG (packet 39765, de-chunked) | MD5 hash generated via `md5sum` |
| Capture file itself | SHA256 hash generated via `sha256sum Exercise.pcapng` |
| Artist search ("r4w", Packet details scope) | Artist 1 = `r4w8173` |
| Packet 12 comment | Instructions pointing back to the JPEG extraction/hash task |
| Export Objects → `.txt` files | `note.txt` — ASCII-art text decoding to **RACEHTOMATEST** |
| Analyze → Expert Information | 1 Warning type ("Illegal characters found in header name," HTTP), occurring 1,636 times; 15 Malformed Packet errors (13 HTTP, 2 JFIF/JPEG) |
| Protocol Hierarchy | 58,620 frames, 99.8% TCP, HTTP = 6.1% of bytes despite 1.9% of packets |
| Conversations (TCP) | Two dominant conversations (54 MB, 55 MB) account for the vast majority of capture volume |

**Expert Info assessment:** the volume and pattern of warnings (1,636
identical-type header warnings, malformed JFIF errors aligning with
the chunked-transfer JPEG reassembly issue) point toward a
synthetically generated or intentionally crafted capture rather than
organic user traffic — consistent with an automated traffic generator,
as expected in a training exercise. In a live investigation, this
pattern would prompt an initial assessment of traffic legitimacy before
any assumption of malicious intent.

---

## 10. Conclusion

The most transferable skill demonstrated across this exercise was not
any single tool feature, but the discipline of **verifying the correct
packet, stream, or finding before acting on it**. Several steps —
stream identification, chunked-versus-raw JPEG extraction, and the
Expert Info assessment — only resolved correctly after re-confirmation
rather than assumption. That habit of cross-checking evidence before
drawing a conclusion is directly applicable to real-world packet-level
triage and incident investigation.

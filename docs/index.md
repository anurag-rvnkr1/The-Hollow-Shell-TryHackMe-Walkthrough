---
layout: default
title: "The Hollow Shell — Premium CTF Write-up"
description: "Portfolio-grade TryHackMe documentation covering reconnaissance, source disclosure, Zip Slip, arbitrary file write, worker-mediated execution, and reverse-shell access."
---

<style>
.hero {
  padding: 2.5rem 2rem;
  margin: 1rem 0 2rem;
  border: 1px solid #d8dee6;
  border-radius: 18px;
  background: linear-gradient(135deg, #142033 0%, #1d3a52 58%, #0b5268 100%);
  color: #fff;
  box-shadow: 0 14px 40px rgba(20, 32, 51, 0.16);
}
.hero h1 {
  color: #fff !important;
  border: 0 !important;
  margin: 0 0 .65rem;
  font-size: clamp(2.3rem, 6vw, 4.4rem);
  letter-spacing: -.035em;
}
.hero p {
  color: #dce7ef;
  max-width: 850px;
  font-size: 1.05rem;
}
.hero .meta {
  color: #ddb146;
  font-weight: 700;
  letter-spacing: .08em;
  text-transform: uppercase;
  font-size: .82rem;
}
.pills {
  display: flex;
  flex-wrap: wrap;
  gap: .5rem;
  margin: 1.25rem 0 0;
}
.pill {
  display: inline-block;
  padding: .35rem .75rem;
  border-radius: 999px;
  background: rgba(255,255,255,.11);
  border: 1px solid rgba(255,255,255,.18);
  color: #fff;
  font-size: .82rem;
  font-weight: 650;
}
.callout {
  border-left: 4px solid #4dc1db;
  padding: 1rem 1.15rem;
  margin: 1.25rem 0;
  background: #f2f8fa;
  border-radius: 0 10px 10px 0;
}
.warning {
  border-left-color: #ddb146;
  background: #fff9eb;
}
.evidence {
  margin: 1.5rem 0 2.25rem;
  text-align: center;
}
.evidence img {
  max-width: 100%;
  border-radius: 12px;
}
.evidence figcaption {
  margin-top: .55rem;
  color: #667085;
  font-size: .85rem;
}
.metric-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(155px, 1fr));
  gap: .75rem;
  margin: 1.25rem 0 1.75rem;
}
.metric {
  padding: 1rem;
  border: 1px solid #d8dee6;
  border-radius: 12px;
  background: #fff;
}
.metric strong {
  display: block;
  font-size: 1.25rem;
  color: #142033;
}
.metric span {
  color: #667085;
  font-size: .85rem;
}
.chain {
  padding: 1.25rem;
  border-radius: 14px;
  background: #0f1626;
  color: #e7edf4;
  overflow-x: auto;
}
.section-kicker {
  color: #0b7285;
  text-transform: uppercase;
  letter-spacing: .1em;
  font-size: .75rem;
  font-weight: 800;
}
</style>

<div class="hero">

<div class="meta">TryHackMe · Hacker Holidays 2026 · Day 10</div>

# 🐚 The Hollow Shell

<p><strong>From an ordinary ZIP upload to a filesystem trust-boundary failure.</strong></p>

<p>
A portfolio-grade technical investigation of the Byte Lotus Shoreline Display portal,
covering reconnaissance, source disclosure, unsafe archive extraction, Zip Slip,
arbitrary file write, background-worker abuse, and controlled shell access.
</p>

<div class="pills">
<span class="pill">Web Security</span>
<span class="pill">Zip Slip</span>
<span class="pill">Path Traversal</span>
<span class="pill">Arbitrary File Write</span>
<span class="pill">Worker Analysis</span>
<span class="pill">Reverse Shell</span>
<span class="pill">Linux</span>
</div>

</div>

<div class="callout warning">

### 🔐 Public-portfolio policy

The final challenge flag is **intentionally redacted** throughout this documentation. Target-specific addresses are normalized to placeholders where appropriate.

This page documents the **reasoning, evidence, exploitation chain, root cause, and defensive lessons** rather than publishing the challenge answer as an answer key.

</div>

## 🎯 Challenge at a Glance

<div class="metric-grid">
<div class="metric"><strong>Day 10</strong><span>Hacker Holidays 2026</span></div>
<div class="metric"><strong>Web</strong><span>Challenge category</span></div>
<div class="metric"><strong>Medium</strong><span>Room difficulty</span></div>
<div class="metric"><strong>Zip Slip</strong><span>Primary weakness</span></div>
<div class="metric"><strong>File Write</strong><span>Core primitive</span></div>
<div class="metric"><strong>RCE</strong><span>Final impact</span></div>
</div>

| Property | Value |
|---|---|
| **Platform** | TryHackMe |
| **Room** | The Hollow Shell |
| **Event** | Hacker Holidays 2026 |
| **Day** | 10 |
| **Category** | Web |
| **Difficulty** | Medium |
| **Primary weakness** | Zip Slip / Path Traversal |
| **Core impact** | Arbitrary File Write |
| **Execution mechanism** | Background worker / hook processing |
| **Shell context** | `roomservice` |
| **Objective** | `/home/roomservice/flag.txt` |
| **Flag** | 🔒 Redacted |

The official room describes the challenge as a web task in which a hotel display portal accepts uploaded “shell” packages and asks the player to find the flag. citeturn0search1

---

# 🧭 Executive Summary

The Hollow Shell is a strong example of **vulnerability chaining**.

The initial application appears to provide a conventional ZIP upload feature. Investigation reveals that the portal:

1. exposes a web application on TCP/5000;
2. contains a discoverable `/upload` endpoint;
3. returns `405 Method Not Allowed` when queried with the wrong method;
4. exposes lab staff credentials in page source;
5. provides an authenticated ZIP upload interface;
6. extracts attacker-controlled archive members;
7. fails to enforce a safe extraction boundary;
8. allows a Zip Slip traversal to become an arbitrary file write;
9. exposes a background-worker / automation-hook trust boundary;
10. allows the file-write primitive to reach an execution-relevant location;
11. ultimately provides shell access as `roomservice`.

The security lesson is larger than the individual vulnerability:

> **An arbitrary file-write primitive becomes significantly more dangerous when attacker-controlled files can reach a trusted automated consumer.**

---

# 🗺️ Attack Path

<div class="chain">

```text
┌───────────────────────────┐
│  01  Reconnaissance       │
│  Nmap → 22 / 5000         │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  02  Web Enumeration      │
│  /upload → HTTP 405       │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  03  Source Inspection    │
│  Lab credential disclosed │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  04  Staff Dashboard      │
│  ZIP + worker clue        │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  05  Baseline Upload      │
│  shells/<random-id>/      │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  06  Zip Slip             │
│  Path traversal           │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  07  Arbitrary File Write │
│  Marker + static proof    │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  08  Worker Hook          │
│  Trusted consumer         │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  09  Controlled Callback  │
│  Worker-mediated exec     │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  10  Reverse Shell        │
│  roomservice              │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  11  Objective            │
│  flag.txt — REDACTED      │
└───────────────────────────┘
```

</div>

---

# 🖼️ Evidence Gallery

The portfolio intentionally uses **11 major evidence assets**. Each image represents a meaningful stage of the investigation rather than repetitive command output.

| # | Evidence | What it proves |
|---:|---|---|
| 01 | Room overview | Scope and challenge context |
| 02 | Nmap | Exposed network services |
| 03 | Upload method | POST-only `/upload` clue |
| 04 | Source disclosure | Lab credential discovery |
| 05 | Dashboard | ZIP workflow and worker clue |
| 06 | Test upload | Normal extraction behavior |
| 07 | Zip Slip marker | Traversal confirmed |
| 08 | Static proof | Arbitrary file write confirmed |
| 09 | Hook placement | Worker execution path |
| 10 | Payload archive | Final archive structure |
| 11 | Shell / objective | Post-exploitation evidence |

---

# 01 — Room Overview

<figure class="evidence">

<img src="assets/01-cover.svg" alt="The Hollow Shell TryHackMe room overview">

<figcaption><strong>Figure 01.</strong> Challenge scope and exploitation theme.</figcaption>

</figure>

The room belongs to the **Hacker Holidays 2026** sequence and is categorized as a **Medium Web** challenge. The official room identifies the objective simply as finding the flag. citeturn0search1

The central application is the Byte Lotus hotel's **Shoreline Display** portal.

---

# 02 — Reconnaissance

## Service Discovery

The first step is to establish the exposed attack surface.

```bash
nmap TARGET_IP
```

The important services are:

```text
22/tcp    open    ssh
5000/tcp  open    web application
```

<figure class="evidence">

<img src="assets/02-recon-nmap.svg" alt="Nmap reconnaissance showing ports 22 and 5000">

<figcaption><strong>Figure 02.</strong> Initial service enumeration.</figcaption>

</figure>

The application on TCP/5000 becomes the primary target.

### Analyst observation

The presence of SSH is useful reconnaissance information, but it is not required for the main exploitation chain documented here.

---

# 03 — Discovering the Upload Endpoint

Web-content enumeration identifies an interesting route:

```text
/upload
```

Requesting it directly with `GET` produces:

```text
405 Method Not Allowed
```

<figure class="evidence">

<img src="assets/03-upload-method.svg" alt="Upload endpoint returning HTTP 405">

<figcaption><strong>Figure 03.</strong> HTTP method analysis reveals a POST-only upload surface.</figcaption>

</figure>

A `405` is valuable because it indicates that the route exists but does not accept the method used.

Conceptually:

```text
404
└── resource not found

405
└── resource exists
    └── requested HTTP method rejected
```

This points toward a likely upload `POST` endpoint.

---

# 04 — Source Disclosure

The login page source contains a lab-specific HTML comment exposing the initial staff credential.

<figure class="evidence">

<img src="assets/04-source-disclosure.svg" alt="Login page source showing lab credential disclosure">

<figcaption><strong>Figure 04.</strong> Client-visible source exposes the challenge's seeded staff credential.</figcaption>

</figure>

The lab credential used during the investigation is:

```text
concierge : StayNoticed2024!
```

### Security significance

Anything delivered to the browser should be considered visible to the client.

That includes:

- HTML comments;
- JavaScript;
- hidden fields;
- source maps;
- debug information;
- embedded configuration.

Secrets should remain server-side.

---

# 05 — Shoreline Display Dashboard

Authentication exposes the Shoreline Display staff interface.

<figure class="evidence">

<img src="assets/05-dashboard-upload.svg" alt="Shoreline Display dashboard with ZIP upload functionality">

<figcaption><strong>Figure 05.</strong> Authenticated ZIP upload functionality and the background-worker clue.</figcaption>

</figure>

The application expects uploaded shells to contain:

```text
shell.json
```

It also states that shells may contain:

```text
automation hooks
```

which are applied by a:

```text
theme worker
```

This becomes the most important architectural clue in the room.

---

# 06 — Establishing Normal Upload Behavior

Before testing the extraction boundary, a normal ZIP is uploaded.

The manifest follows the expected structure:

```json
{
  "name": "test",
  "assets": []
}
```

<figure class="evidence">

<img src="assets/06-test-upload.svg" alt="Normal shell ZIP upload">

<figcaption><strong>Figure 06.</strong> Baseline shell upload and application storage behavior.</figcaption>

</figure>

The application stores the uploaded shell below a per-upload directory similar to:

```text
/shells/<random-id>/
```

This gives us an expected filesystem boundary.

---

# 07 — Zip Slip Discovery

## The Vulnerability

The archive extraction behavior does not adequately constrain attacker-controlled member paths.

A traversal entry such as:

```text
../marker.txt
```

can escape the intended extraction directory.

<figure class="evidence">

<img src="assets/07-zipslip-marker.svg" alt="Zip Slip traversal marker proof">

<figcaption><strong>Figure 07.</strong> Harmless traversal confirms that archive members can escape their expected extraction directory.</figcaption>

</figure>

### Security model

```text
Expected:

/shells/abc123/file.txt


Traversal:

/shells/abc123/../marker.txt


Resolved:

/shells/marker.txt
```

This confirms the presence of a **Zip Slip / path traversal** condition.

---

# 08 — Arbitrary File Write Proof

A second validation places a controlled static asset outside the shell directory.

The purpose is to prove that the vulnerability provides more than a cosmetic path anomaly.

<figure class="evidence">

<img src="assets/08-zipslip-static-proof.svg" alt="Static file write confirmation">

<figcaption><strong>Figure 08.</strong> Browser-readable static content confirms arbitrary file write.</figcaption>

</figure>

The resulting chain is:

```text
Attacker-controlled ZIP
          ↓
Traversal-bearing filename
          ↓
Unsafe extraction
          ↓
Filesystem boundary bypass
          ↓
Arbitrary file write
          ↓
Browser-readable artifact
```

This staged validation is important because it proves the primitive independently before attempting to turn it into code execution.

---

# 09 — Identifying the Worker Hook

The dashboard's mention of automation hooks provides the next lead.

<figure class="evidence">

<img src="assets/09-hook-write.svg" alt="Worker hook file placement">

<figcaption><strong>Figure 09.</strong> Traversal reaches the worker-controlled hook location.</figcaption>

</figure>

The exploitation hypothesis becomes:

```text
Zip Slip
   ↓
Write controlled file
   ↓
Place file in worker hook location
   ↓
Worker processes hook
   ↓
Controlled execution
```

This is where the impact of the arbitrary-write vulnerability increases substantially.

---

# 10 — Payload Archive

The final controlled archive contains the required manifest and the callback under the traversal path.

<figure class="evidence">

<img src="assets/10-payload-archive.svg" alt="Final reverse-shell ZIP archive structure">

<figcaption><strong>Figure 10.</strong> Final archive structure used to reach the worker-controlled callback location.</figcaption>

</figure>

Conceptually:

```text
reverse-shell.zip
├── shell.json
└── ../../hooks/callback.py
```

For public documentation, the callback address is represented as:

```text
YOUR_ATTACKER_IP
```

The important operational rule is:

> Build the archive only after the callback address has been correctly configured.

---

# 11 — Shell Access & Objective

The worker processes the planted callback and the listener receives a shell in the application context.

The resulting user is:

```text
roomservice
```

The challenge objective is located at:

```text
/home/roomservice/flag.txt
```

<figure class="evidence">

<img src="assets/11-shell-flag-redacted.svg" alt="Reverse shell with flag output redacted">

<figcaption><strong>Figure 11.</strong> Post-exploitation shell evidence. The final flag output is intentionally redacted.</figcaption>

</figure>

The public portfolio version deliberately does **not** display the flag.

---

# 🔬 Technical Analysis

## Zip Slip

Zip Slip occurs when archive member paths are trusted as safe filesystem paths.

The dangerous pattern is:

```text
Extraction Root
      +
Untrusted Archive Filename
      ↓
Destination Path
```

without canonicalizing and validating the final destination.

A secure extractor must ensure:

```text
canonical(destination) ∈ extraction_root
```

before writing.

---

## Arbitrary File Write

The Zip Slip vulnerability transforms the upload feature into an arbitrary file-write primitive.

The security boundary changes from:

```text
"User can upload a shell"
```

to:

```text
"User can influence arbitrary writable filesystem locations"
```

subject to the permissions of the application process.

---

## Worker-Mediated Execution

The dashboard's automation-worker functionality creates a second trust boundary.

The application effectively has:

```text
Untrusted Input
      ↓
Filesystem
      ↓
Trusted Worker
```

If the worker processes attacker-controlled source files, a filesystem-write primitive can become code execution.

---

# 🧩 Exploitation Chain

```text
Source Disclosure
       │
       ▼
Authenticated Access
       │
       ▼
ZIP Upload
       │
       ▼
Unsafe Archive Extraction
       │
       ▼
Zip Slip
       │
       ▼
Arbitrary File Write
       │
       ▼
Worker Hook Location
       │
       ▼
Worker Processes Callback
       │
       ▼
Code Execution
       │
       ▼
Reverse Shell
       │
       ▼
roomservice
```

The critical relationship is:

```text
File Write
    +
Trusted Automated Consumer
    =
Code Execution
```

---

# 🧪 Investigation Methodology

The room is best understood as a sequence of progressively stronger proofs.

### Stage 1 — Discover

```text
Find services
Find web routes
Find authentication surface
```

### Stage 2 — Understand

```text
Inspect source
Authenticate
Read application hints
Map upload behavior
```

### Stage 3 — Validate

```text
Normal upload
↓
Harmless traversal
↓
Static write
```

### Stage 4 — Chain

```text
Worker hook
↓
Controlled callback
↓
Shell
```

### Stage 5 — Document

```text
Evidence
↓
Root cause
↓
Impact
↓
Remediation
```

This approach makes the exploitation process reproducible and defensible.

---

# 🛡️ Root Cause

The central vulnerability is insufficient validation of archive member paths.

The application trusts a path supplied by the ZIP archive instead of enforcing:

```text
archive member
      ↓
canonical path
      ↓
safe extraction root
```

The secondary design issue is allowing a background worker to process content that can be influenced by the upload mechanism.

---

# 🔧 Remediation

## 1. Enforce extraction-root boundaries

Every archive member should be canonicalized and checked before writing.

Reject:

```text
../
..\ 
absolute paths
```

and unsafe symlink behavior.

---

## 2. Separate uploads from executable code

Uploaded content should reside in a dedicated, non-executable storage location.

Avoid:

```text
uploads → application code → worker
```

Prefer:

```text
uploads → isolated data store → validated data processing
```

---

## 3. Never execute uploaded source

Worker automation should process a strict data format rather than executing arbitrary files supplied through an upload.

---

## 4. Apply least privilege

The web application and worker should run with minimal filesystem and process permissions.

---

## 5. Remove client-visible secrets

Credentials must never be embedded in HTML comments, JavaScript, or other browser-delivered resources.

---

## 6. Monitor filesystem boundaries

Alert when the application writes outside its expected upload directory.

---

# 🕵️ Detection Opportunities

| Signal | Possible meaning |
|---|---|
| `../` in ZIP members | Zip Slip attempt |
| Absolute archive paths | Extraction-boundary bypass attempt |
| Writes outside upload root | Arbitrary file-write activity |
| New `.py` files in worker directories | Possible execution staging |
| Web process spawning shell interpreters | Potential RCE |
| Unexpected outbound TCP connections | Possible reverse shell |
| Worker creates unexpected child processes | Potential compromise |
| Application files modified after upload | Possible archive traversal |

---

# 📊 Security Impact

| Component | Finding | Impact |
|---|---|---|
| Login page | Source disclosure | Credential exposure |
| ZIP extraction | Zip Slip | Path traversal |
| Filesystem | Arbitrary write | Attacker-controlled file placement |
| Worker | Hook processing | Execution boundary crossed |
| Host | Reverse shell | Remote command execution |
| User context | `roomservice` | Application-level shell |
| Objective | `flag.txt` | Challenge completion |

---

# 🧠 Lessons Learned

## 01 — HTTP errors can reveal application structure

A `405` response can confirm an endpoint exists even when the request method is incorrect.

## 02 — Source is part of the attack surface

Anything delivered to a browser can be inspected.

## 03 — Archive filenames are untrusted input

ZIP extraction should be treated as filesystem-sensitive processing.

## 04 — Validate primitives incrementally

A harmless marker is a better first proof than immediately deploying an execution payload.

## 05 — Impact depends on architecture

Arbitrary file write becomes much more severe when a trusted process automatically consumes the written file.

## 06 — Application hints can reveal trust boundaries

The worker description provided a clue about where attacker-controlled content could cross into trusted execution.

## 07 — Good CTF documentation explains why

A professional write-up should answer:

```text
What did I discover?
Why did it matter?
What did I test?
What did the result prove?
How did the vulnerabilities chain?
How would a defender fix it?
```

---

# 🧰 Skills & Techniques

<div class="pills">
<span class="pill">Nmap</span>
<span class="pill">Gobuster</span>
<span class="pill">HTTP Analysis</span>
<span class="pill">HTML Source Inspection</span>
<span class="pill">ZIP Analysis</span>
<span class="pill">Python</span>
<span class="pill">Path Traversal</span>
<span class="pill">Zip Slip</span>
<span class="pill">Arbitrary File Write</span>
<span class="pill">Netcat</span>
<span class="pill">Linux</span>
<span class="pill">Web Exploitation</span>
</div>

---

# 📁 Evidence & Documentation Structure

```text
The-Hollow-Shell-TryHackMe-Walkthrough/
│
├── README.md
│
├── Documentation/
│   ├── Documentation.md
│   └── Documentation.docx
│
├── Resources/
│   └── notes.md
│
├── Screenshots/
│   └── README.md
│
├── docs/
│   ├── index.md
│   │
│   └── assets/
│       ├── 01-cover.svg
│       ├── 02-recon-nmap.svg
│       ├── 03-upload-method.svg
│       ├── 04-source-disclosure.svg
│       ├── 05-dashboard-upload.svg
│       ├── 06-test-upload.svg
│       ├── 07-zipslip-marker.svg
│       ├── 08-zipslip-static-proof.svg
│       ├── 09-hook-write.svg
│       ├── 10-payload-archive.svg
│       ├── 11-shell-flag-redacted.svg
│       └── css/
│           └── custom.scss
│
└── _config.yml
```

The image set is deliberately limited to **11 major evidence assets** so the portfolio remains readable and visually consistent.

---

# 🖼️ Complete Asset Index

## `01-cover.svg`

Challenge identity, theme, and high-level exploitation focus.

## `02-recon-nmap.svg`

Network reconnaissance showing the exposed services.

## `03-upload-method.svg`

The `/upload` endpoint returning `405 Method Not Allowed`.

## `04-source-disclosure.svg`

Client-visible source containing the lab credential.

## `05-dashboard-upload.svg`

Authenticated Shoreline Display upload interface.

## `06-test-upload.svg`

Baseline ZIP upload and normal application behavior.

## `07-zipslip-marker.svg`

Harmless traversal marker proving the extraction boundary is bypassable.

## `08-zipslip-static-proof.svg`

Static-file validation of arbitrary file write.

## `09-hook-write.svg`

Controlled write reaching the worker hook location.

## `10-payload-archive.svg`

Final archive structure used in the execution chain.

## `11-shell-flag-redacted.svg`

Reverse shell evidence with the objective output redacted.

---

# 📚 Related Documentation

### Full Technical Report

**[`Documentation/Documentation.md`](../Documentation/Documentation.md)**

The detailed technical report contains the complete investigation, exploitation reasoning, evidence mapping, remediation, and detection analysis.

### Research Notes

**[`Resources/notes.md`](../Resources/notes.md)**

The consolidated research and technical notes contain the underlying reasoning, observations, quick reference, troubleshooting, and defensive analysis.

### Word Report

**[`Documentation/Documentation.docx`](../Documentation/Documentation.docx)**

A formatted Word version of the technical report.

### Repository README

**[`README.md`](../README.md)**

The repository landing page provides the shorter portfolio overview.

---

# 🏆 Portfolio Takeaway

The most important result of this room is not simply obtaining a shell.

The meaningful security workflow is:

```text
Enumerate
    ↓
Understand the application
    ↓
Identify trust boundaries
    ↓
Establish baseline behavior
    ↓
Validate the vulnerability safely
    ↓
Demonstrate impact
    ↓
Chain the primitive
    ↓
Collect evidence
    ↓
Explain root cause
    ↓
Recommend remediation
```

The Hollow Shell demonstrates how a seemingly ordinary upload feature can become a remote-code-execution path when:

```text
untrusted archive paths
        +
unsafe extraction
        +
arbitrary file write
        +
trusted worker processing
```

are allowed to interact.

That combination is the core lesson of the challenge.

---

# 🔐 Flag Policy

The final flag is intentionally excluded from this portfolio documentation.

Public evidence stops at:

```text
/home/roomservice/flag.txt
```

and represents the output as:

```text
[ FLAG OUTPUT REDACTED ]
```

This keeps the repository focused on security methodology and technical understanding rather than functioning as a direct answer repository.

---

# ⚖️ Responsible Use

This documentation concerns an intentionally vulnerable **TryHackMe** training environment.

The techniques described should only be reproduced against:

- systems you own;
- systems for which you have explicit authorization;
- intentionally vulnerable training environments.

Do not apply these techniques to production systems without authorization.

---

# 🔗 References

- [TryHackMe — The Hollow Shell](https://tryhackme.com/room/hh-thehollowshell-ddb582ac)
- [OWASP — Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [OWASP — File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [Reference walkthrough — Crystalcascade14](https://crystalcascade14.medium.com/the-hollow-shell-9d9c1946d6f2)

The TryHackMe room identifies **The Hollow Shell** as a 90-point, Medium-difficulty Web challenge in Hacker Holidays 2026. citeturn0search1

Independent public write-ups also describe the core challenge around unsafe ZIP extraction / Zip Slip and a subsequent execution path, providing useful cross-checking of the general vulnerability chain. citeturn0search0turn0search2

---

# 👤 Author

## Anurag Revankar

**Cybersecurity · Web Security · CTFs · Vulnerability Research**

<p align="center">

<a href="https://github.com/anurag-rvnkr1">
<img src="https://img.shields.io/badge/GitHub-anurag--rvnkr1-181717?style=for-the-badge&logo=github&logoColor=white" alt="GitHub">
</a>

</p>

---

<div class="callout">

### 🐚 Final Note

**The Hollow Shell**

```text
Enumerate
    →
Understand
    →
Validate
    →
Exploit
    →
Document
    →
Defend
```

A good CTF write-up should not only show **how the target was compromised**.

It should explain **why the vulnerability existed, how the weaknesses chained together, what evidence proved each stage, and how the system could be made resilient against the same attack.**

</div>

<p align="center">
  <sub>TryHackMe Hacker Holidays 2026 · Day 10 · Public portfolio edition · Flags intentionally redacted</sub>
</p>

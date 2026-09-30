# 🐚 The Hollow Shell — TryHackMe Walkthrough

<p align="center">
  <img src="docs/assets/01-cover.svg" alt="The Hollow Shell — TryHackMe" width="900">
</p>

<p align="center">
  <strong>Hacker Holidays 2026 · Day 10 · Web Security</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TryHackMe-Hacker%20Holidays%202026-red?style=for-the-badge" alt="TryHackMe">
  <img src="https://img.shields.io/badge/Category-Web-0f9d58?style=for-the-badge" alt="Web">
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium">
  <img src="https://img.shields.io/badge/Focus-Zip%20Slip-blue?style=for-the-badge" alt="Zip Slip">
  <img src="https://img.shields.io/badge/Status-Completed-success?style=for-the-badge" alt="Completed">
</p>

---

## 📌 About This Repository

This repository contains a **portfolio-oriented, original technical walkthrough** of **TryHackMe — The Hollow Shell**, a Hacker Holidays 2026 web-security challenge.

The room revolves around a deceptively simple ZIP upload workflow used by the Byte Lotus Hotel's **Shoreline Display** service. The investigation develops into an exploitation chain involving:

```text
Reconnaissance
      │
      ▼
Web service discovery
      │
      ▼
Hidden POST-only upload endpoint
      │
      ▼
HTML source disclosure
      │
      ▼
Authenticated Shoreline Display portal
      │
      ▼
ZIP upload analysis
      │
      ▼
Zip Slip / path traversal
      │
      ▼
Arbitrary file write
      │
      ▼
Background-worker hook execution
      │
      ▼
Reverse shell
      │
      ▼
roomservice shell
      │
      ▼
Flag location
```

The official room describes **The Hollow Shell** as a **Web** challenge with **Medium** difficulty and **90 points**. citeturn0search1

> **⚠️ Flag policy:** The final flag is intentionally **redacted** throughout this public repository. The goal is to document the methodology and technical reasoning without publishing the challenge answer verbatim.

---

## 🎯 Challenge Information

| Property | Details |
|---|---|
| Platform | TryHackMe |
| Event | Hacker Holidays 2026 |
| Day | 10 |
| Room | **The Hollow Shell** |
| Category | Web |
| Difficulty | Medium |
| Points | 90 |
| Primary vulnerability | Zip Slip |
| Secondary impact | Arbitrary file write |
| Execution primitive | Background worker / hook processing |
| Final access | Reverse shell |
| Flag | 🔒 Redacted |

🔗 **Room:** [TryHackMe — The Hollow Shell](https://tryhackme.com/room/hh-thehollowshell-ddb582ac)

---

## 🧠 Executive Summary

The target exposes a web application on TCP port `5000`. Enumeration reveals a login page and an upload endpoint that responds differently depending on the HTTP method.

Inspection of the login page source reveals credentials embedded in an HTML comment. After authentication, the Shoreline Display dashboard exposes a ZIP-based upload mechanism and, importantly, mentions an **automation hook** processed by a background worker.

A benign archive establishes the application's extraction behavior. A crafted ZIP archive then demonstrates a **Zip Slip** condition: archive members containing traversal sequences can escape the intended extraction directory.

The arbitrary-write primitive is first validated using a harmless marker and then through a browser-readable static asset. Once the application's worker-controlled `hooks/` location is identified, the same primitive can be used to place a controlled Python callback there. The worker processes the planted hook, resulting in a reverse shell running in the application context.

The final objective file is located under the `roomservice` user's home directory. Its value is intentionally omitted from this repository.

---

## 🔗 Attack Path

```text
┌──────────────────────┐
│  Target Enumeration  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ TCP/5000 Web Service │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ /upload → HTTP 405   │
│ POST-only endpoint   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Source Inspection    │
│ Credential Disclosure│
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Shoreline Dashboard  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ ZIP Upload Workflow  │
│ + Theme Worker Hint   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Zip Slip             │
│ ../ path traversal   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Arbitrary File Write │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ hooks/ Worker Path   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Python Callback      │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Reverse Shell        │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Flag File — Redacted │
└──────────────────────┘
```

---

## 🔎 Methodology

### 1. Reconnaissance

The first stage establishes the exposed attack surface.

A service scan identifies:

```text
22/tcp    open    ssh
5000/tcp  open    web application
```

The web service becomes the primary focus because it exposes the Shoreline Display application.

### 2. Web Enumeration

Directory enumeration identifies an `/upload` route. Requesting it directly with a browser produces:

```text
405 Method Not Allowed
```

Rather than treating the response as a dead end, the status code provides useful information: the route exists, but the current HTTP method is not accepted.

This suggests a POST-oriented upload endpoint.

### 3. Source Inspection

The login page contains an HTML comment with lab-specific staff credentials.

This is an important lesson in web reconnaissance:

> Client-delivered HTML is part of the application's attack surface and should be inspected before assuming the login form itself is the only source of information.

After authenticating, the dashboard exposes the Shoreline Display upload workflow.

### 4. Understanding the Upload Mechanism

The dashboard accepts a ZIP archive containing a `shell.json` manifest.

The interface also mentions an **automation hook** that is processed by a **theme worker** shortly after a shell is uploaded.

That seemingly informational sentence becomes the bridge between:

```text
File upload
```

and

```text
Server-side execution
```

### 5. Baseline Upload

Before attempting exploitation, a normal ZIP package is uploaded.

The application creates a per-upload directory resembling:

```text
shells/<random-id>/
```

This establishes the expected extraction boundary.

The next question is therefore:

> Can an archive member escape that directory during extraction?

---

## 💥 Zip Slip

**Zip Slip** is an archive extraction vulnerability in which attacker-controlled filenames contain path traversal sequences such as:

```text
../
```

If an application concatenates the extraction directory with an archive member's filename without validating the resulting canonical path, an attacker may cause files to be written outside the intended directory.

Conceptually:

```text
Expected:

/app/shells/<id>/file.txt

Malicious archive member:

../../static/proof.css

Result:

/app/static/proof.css
```

The vulnerability is not the ZIP format itself. The security failure occurs when the application trusts archive entry paths during extraction.

---

## 🧪 Safe Vulnerability Validation

The exploit chain is deliberately validated in stages.

### Stage A — Harmless marker

A ZIP archive is created with a traversal entry targeting a harmless marker file.

The successful placement outside the expected shell directory demonstrates that archive extraction does not enforce a strict filesystem boundary.

### Stage B — Browser-readable static file

A second validation targets the application's static directory.

The resulting file can be requested through the web server, providing a clean, non-destructive confirmation of arbitrary file write.

The repository's generated evidence plate records this validation without exposing unnecessary challenge secrets.

---

## ⚙️ From File Write to Code Execution

Arbitrary file write alone does not automatically mean code execution.

The dashboard provides the missing clue:

```text
automation hooks
+
background theme worker
```

The investigation therefore looks for a worker-consumed hook location.

The upload primitive is used to target the application's `hooks/` directory. A controlled Python callback is then placed there.

The resulting chain becomes:

```text
ZIP traversal
      ↓
Write attacker-controlled Python file
      ↓
File lands in worker-controlled hooks/
      ↓
Theme worker processes hook
      ↓
Python callback executes
      ↓
Outbound connection to listener
      ↓
Interactive shell
```

This is the key escalation in the challenge.

---

## 🐚 Reverse Shell

A listener is prepared on the authorized AttackBox:

```bash
nc -lvnp 4444
```

The callback establishes an outbound TCP connection and attaches the socket to a shell process.

For reusable documentation, the attacker address is represented as:

```text
YOUR_ATTACKER_IP
```

rather than publishing a hard-coded lab address.

After the worker processes the planted hook, the listener receives a shell in the application context.

The resulting prompt identifies the effective user as:

```text
roomservice
```

---

## 🚩 Flag

The final flag is stored under:

```text
/home/roomservice/flag.txt
```

The actual flag value is **intentionally redacted** from this repository.

This is a deliberate documentation choice to preserve the learning value of the write-up while avoiding publication of the challenge answer.

---

## 🧩 Vulnerability Analysis

### Primary vulnerability

**CWE-22 — Improper Limitation of a Pathname to a Restricted Directory ('Path Traversal')**

The ZIP extractor fails to ensure that every extracted file remains within the intended extraction directory.

### Impact

The vulnerable extraction behavior provides:

- arbitrary file creation
- arbitrary file overwrite where permissions allow
- access to application directories outside the upload root
- potential modification of executable or template files
- code execution when a writable location is consumed by a worker or interpreter

### Why the worker matters

The background worker turns a filesystem primitive into an execution primitive.

Without a process consuming attacker-controlled files, the vulnerability may remain limited to file write.

With a worker automatically executing files from a reachable directory, the impact becomes substantially greater.

---

## 🛡️ Defensive Recommendations

A production application should treat uploaded archives as **untrusted input**.

### 1. Canonicalize extraction paths

For every archive member:

```text
candidate = extraction_root + archive_member_name
```

Resolve the candidate path and verify that it remains inside the intended extraction directory.

### 2. Reject traversal components

Reject archive entries containing traversal patterns that escape the extraction root.

Do not rely solely on string matching; canonical-path validation is required.

### 3. Separate uploads from executable code

Uploaded files should never share a writable directory with:

- Python modules
- application templates
- worker hooks
- executable scripts
- server-side configuration

### 4. Disable automatic execution of uploaded content

A background worker should process data as data, not execute newly uploaded source code.

### 5. Remove secrets from source

Credentials should never be embedded in HTML comments, templates, JavaScript, or other client-delivered content.

### 6. Apply least privilege

The web process and background worker should have the minimum filesystem permissions necessary for their functions.

### 7. Monitor suspicious archive entries

Security monitoring should flag archive members containing traversal sequences and unexpected writes to application code directories.

---

## 🧰 Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Service and port enumeration |
| `gobuster` | Web content discovery |
| Browser DevTools / View Source | Source inspection |
| Python 3 | ZIP construction and testing |
| `zipfile` | Controlled archive generation |
| `unzip` | Archive inspection |
| `nc` / Netcat | Reverse-shell listener |
| TryHackMe AttackBox | Authorized testing environment |

---

## 📸 Evidence

The public documentation uses a small, numbered evidence set rather than flooding the repository with screenshots.

```text
docs/assets/
├── 01-cover.svg
├── 02-recon-nmap.svg
├── 03-upload-method.svg
├── 04-source-disclosure.svg
├── 05-dashboard-upload.svg
├── 06-test-upload.svg
├── 07-zipslip-marker.svg
├── 08-zipslip-static-proof.svg
├── 09-hook-write.svg
├── 10-payload-archive.svg
└── 11-shell-flag-redacted.svg
```

The final evidence image intentionally contains **redacted flag information**.

---

## 📁 Repository Structure

```text
The-Hollow-Shell-TryHackMe-Walkthrough/
│
├── .github/
│   └── workflows/
│       └── pages.yml
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
│   ├── assets/
│   │   ├── 01-cover.svg
│   │   ├── 02-recon-nmap.svg
│   │   ├── 03-upload-method.svg
│   │   ├── 04-source-disclosure.svg
│   │   ├── 05-dashboard-upload.svg
│   │   ├── 06-test-upload.svg
│   │   ├── 07-zipslip-marker.svg
│   │   ├── 08-zipslip-static-proof.svg
│   │   ├── 09-hook-write.svg
│   │   ├── 10-payload-archive.svg
│   │   ├── 11-shell-flag-redacted.svg
│   │   └── css/
│   │       ├── custom.scss
│   │       └── style.scss
│   │
│   └── index.md
│
├── CONTRIBUTING.md
├── SECURITY.md
├── README.md
└── _config.yml
```

---

## 📚 Learning Outcomes

This room demonstrates several practical web-security concepts that are useful beyond CTF environments:

- service enumeration
- HTTP method analysis
- source-code disclosure
- authenticated attack-surface mapping
- insecure file upload analysis
- archive extraction security
- Zip Slip
- path traversal
- arbitrary file write
- background-worker abuse
- reverse-shell fundamentals
- filesystem trust boundaries
- secure archive handling
- least-privilege design

The most important lesson is the **attack-chain mindset**:

```text
A low-impact primitive
        +
A useful application behavior
        =
A much larger security impact
```

In this challenge:

```text
Zip Slip
   +
Writable worker hook
   =
Code execution
```

---

## 📝 Methodology Notes

This write-up intentionally emphasizes **reasoning rather than simply listing commands**.

The investigation follows a repeatable penetration-testing workflow:

1. Map the exposed services.
2. Enumerate application routes.
3. Inspect responses for unintended information disclosure.
4. Authenticate where the lab permits it.
5. Understand the application's intended workflow.
6. Establish a benign baseline.
7. Test one security boundary at a time.
8. Confirm the vulnerability with a harmless artifact.
9. Identify the next application-controlled processing stage.
10. Chain the primitives only within the authorized lab.
11. Record evidence.
12. Document remediation.

This makes the write-up useful as both a CTF solution and a compact case study in web application assessment.

---

## 🔐 Responsible Disclosure & Lab Scope

This repository documents an intentionally vulnerable **TryHackMe** environment.

All testing described here was performed against the authorized CTF/lab target.

Do **not** reproduce the techniques against systems, applications, networks, or accounts without explicit authorization.

The repository intentionally excludes:

- the final flag
- unnecessary real-world credentials
- reusable target-specific secrets
- unrelated personal information

---

## 📖 Full Documentation

For the complete technical report:

- **[Full Markdown Documentation](Documentation/Documentation.md)**
- **[GitHub Pages Version](docs/index.md)**
- **[Word Documentation](Documentation/Documentation.docx)**
- **[Research Notes](Resources/notes.md)**

---

## 🔗 References

- [TryHackMe — The Hollow Shell](https://tryhackme.com/room/hh-thehollowshell-ddb582ac)
- [OWASP — Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [OWASP — File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)

---

## 👤 Author

**Anurag Revankar**

Cybersecurity | CTFs | Web Security | Security Research

- GitHub: [@anurag-rvnkr1](https://github.com/anurag-rvnkr1)
- Repository: [The-Hollow-Shell-TryHackMe-Walkthrough](https://github.com/anurag-rvnkr1/The-Hollow-Shell-TryHackMe-Walkthrough)

---

<p align="center">
  <strong>🐚 The shell was hollow. The upload boundary wasn't.</strong>
</p>

<p align="center">
  <sub>Educational CTF documentation · Flags intentionally redacted</sub>
</p>

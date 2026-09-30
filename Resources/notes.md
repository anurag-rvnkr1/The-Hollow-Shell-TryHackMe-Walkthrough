# The Hollow Shell — Research Notes

## Core chain

1. TCP/5000 exposes the Shoreline Display web application.
2. `/upload` exists but rejects browser-style GET requests with HTTP 405.
3. The login source exposes a lab staff credential.
4. The dashboard accepts ZIP-based “shells” containing `shell.json`.
5. The dashboard explicitly mentions automation hooks handled by a background worker.
6. A ZIP member using `../` proves the extractor does not enforce a path boundary.
7. A write into `static/` confirms the file-write primitive through a browser-readable artifact.
8. A write targeting `hooks/` provides the path to worker-mediated execution.
9. The worker executes a Python callback and returns a reverse shell.
10. The flag is stored at `/home/roomservice/flag.txt`; value intentionally redacted.

## Key terminology

- **Zip Slip:** archive extraction flaw where attacker-controlled member paths escape the intended destination directory.
- **Arbitrary file write:** attacker can cause the application to write attacker-controlled content to a chosen filesystem path.
- **Background worker:** a separate process that asynchronously handles or transforms data after upload.
- **Reverse shell:** the target initiates an outbound connection back to the operator, providing an interactive shell.

## Evidence discipline

The public repository deliberately keeps the final flag out of both text and screenshots. Generated evidence plates are used instead of raw screenshots so the write-up remains portfolio-ready and consistent.

## Safe validation order

**marker → readable static proof → execution path → reverse shell**

That sequence demonstrates the vulnerability with minimum impact before escalating to code execution.

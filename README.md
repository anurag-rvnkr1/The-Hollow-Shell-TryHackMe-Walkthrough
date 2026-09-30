# The Hollow Shell — TryHackMe Walkthrough

![The Hollow Shell](docs/assets/01-cover.svg)

A portfolio-grade, original technical write-up for **TryHackMe — The Hollow Shell**, documenting a web exploitation chain in the Byte Lotus Hotel environment: endpoint enumeration → source disclosure → unsafe ZIP extraction → **Zip Slip** path traversal → arbitrary file write → background-worker execution → reverse shell.

> **Flag policy:** flag values are intentionally redacted from this repository and from the published screenshots. This keeps the write-up useful for learning without publishing the final answer verbatim.

## Room at a glance

| Item | Value |
|---|---|
| Platform | TryHackMe |
| Event | Hacker Holidays 2026 |
| Day | 10 |
| Category | Web |
| Difficulty | Medium |
| Objective | Find the flag |
| Primary flaw | Zip Slip / path traversal in archive extraction |
| Final impact | Arbitrary file write + code execution via worker |

## Attack chain

```text
Open services
   ↓
Web portal on :5000
   ↓
405 on /upload → hidden POST endpoint discovered
   ↓
HTML source disclosure → staff login
   ↓
Dashboard → ZIP upload + theme worker clue
   ↓
Zip Slip → write outside extraction directory
   ↓
Write into static/ → browser-visible proof
   ↓
Write into hooks/ → worker execution path
   ↓
Reverse shell callback
   ↓
roomservice shell → /home/roomservice/flag.txt
```

## Documentation

The complete walkthrough is available here:

- [GitHub Pages write-up](docs/index.md)
- [Full Markdown report](Documentation/Documentation.md)
- [Word report](Documentation/Documentation.docx)
- [Research notes](Resources/notes.md)

## Evidence set

The documentation uses numbered, generated evidence plates stored under `docs/assets/`. The final flag screenshot is deliberately redacted.

## Defensive takeaways

The central lesson is that archive extraction is a trust boundary. A filename such as `../../hooks/callback.py` is data, not a harmless relative path. Extraction code must canonicalize and validate every member path before writing to disk, and background workers should never execute newly uploaded files from attacker-controlled locations.

## Ethical use

This material is for an authorized TryHackMe lab environment. Do not reproduce the technique against systems you do not own or have explicit permission to test.

## References

- [TryHackMe — The Hollow Shell](https://tryhackme.com/room/hh-thehollowshell-ddb582ac)
- [OWASP — Path Traversal / Zip Slip background](https://owasp.org/www-community/attacks/Path_Traversal)

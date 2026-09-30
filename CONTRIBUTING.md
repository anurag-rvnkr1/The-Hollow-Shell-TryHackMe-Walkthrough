# 🤝 Contributing to The Hollow Shell — TryHackMe Walkthrough

Thank you for your interest in contributing to this documentation project.

This repository is a **security-learning and portfolio documentation project** built around the TryHackMe **The Hollow Shell** challenge. Contributions are welcome when they improve technical accuracy, clarity, accessibility, reproducibility, or the overall quality of the documentation.

> **Lab-only principle:** All techniques, payloads, testing procedures, and exploitation guidance in this repository are intended for authorized security labs such as TryHackMe. Do not apply them to systems without explicit permission.

---

## 📋 Table of Contents

- [Code of Conduct](#-code-of-conduct)
- [What You Can Contribute](#-what-you-can-contribute)
- [Repository Standards](#-repository-standards)
- [Documentation Guidelines](#-documentation-guidelines)
- [Screenshot and Asset Guidelines](#-screenshot-and-asset-guidelines)
- [Security and Sensitive Information](#-security-and-sensitive-information)
- [Flag Redaction Policy](#-flag-redaction-policy)
- [Recommended Workflow](#-recommended-workflow)
- [Commit Message Guidelines](#-commit-message-guidelines)
- [Pull Request Guidelines](#-pull-request-guidelines)
- [Quality Checklist](#-quality-checklist)
- [Reporting Issues](#-reporting-issues)
- [License and Attribution](#-license-and-attribution)

---

## 🧭 Code of Conduct

Contributors are expected to:

- Communicate respectfully and professionally.
- Keep discussions focused on technical and educational improvements.
- Provide constructive feedback.
- Avoid harassment, discrimination, or personal attacks.
- Respect the work and contributions of other researchers and authors.
- Keep all security testing within authorized environments.

---

## 🛠️ What You Can Contribute

Useful contributions include:

### 📚 Documentation

- Correcting technical inaccuracies.
- Improving explanations of Zip Slip and path traversal.
- Clarifying the attack chain.
- Improving Markdown formatting.
- Adding defensive-security explanations.
- Improving navigation and cross-references.
- Fixing spelling, grammar, or broken links.

### 🔬 Technical Improvements

- Correcting commands or examples.
- Improving reproducibility of lab steps.
- Adding safer vulnerability-validation examples.
- Explaining why a particular exploitation step works.
- Adding references to authoritative security resources.

### 🎨 Presentation

- Improving GitHub Pages styling.
- Improving diagrams and architecture visuals.
- Replacing unclear screenshots.
- Improving accessibility through descriptive image alt text.
- Improving the structure of documentation pages.

### 🧹 Repository Maintenance

- Removing obsolete files.
- Fixing broken GitHub Pages configuration.
- Improving repository organization.
- Updating documentation metadata.
- Fixing workflow or build issues.

---

## 📁 Repository Standards

Please preserve the existing organization where possible.

```text
.
├── .github/
│   └── workflows/
├── Documentation/
│   └── Documentation.md
├── Resources/
│   └── notes.md
├── Screenshots/
├── docs/
│   ├── assets/
│   │   ├── css/
│   │   └── *.svg / *.png
│   └── index.md
├── README.md
├── CONTRIBUTING.md
├── SECURITY.md
└── _config.yml
```

Use descriptive filenames and avoid unnecessary duplication.

### Naming conventions

Prefer:

```text
01-cover.svg
02-recon-nmap.svg
03-upload-method.svg
04-source-disclosure.svg
```

Avoid:

```text
image1.png
newfinal.png
final-final2.png
screenshot_latest.png
```

Numbered evidence files make the documentation easier to follow and maintain.

---

## 📝 Documentation Guidelines

Documentation should be:

- Original.
- Technically accurate.
- Concise where possible.
- Detailed where the reasoning matters.
- Easy for another learner to reproduce in the authorized lab.
- Written in clear professional English.
- Consistent with the repository's existing terminology.

### Prefer explaining the reasoning

Instead of documenting only:

```bash
gobuster dir ...
```

Explain what the result means:

```text
The enumeration identified /upload. A normal GET request returned
405 Method Not Allowed, suggesting that the endpoint exists but
expects a different HTTP method.
```

The objective is to document **why a step was performed**, not merely what command was typed.

### Preserve the attack-chain narrative

The primary chain should remain understandable:

```text
Recon
  ↓
Web enumeration
  ↓
Source inspection
  ↓
Authentication
  ↓
ZIP upload analysis
  ↓
Zip Slip
  ↓
Arbitrary file write
  ↓
Hook processing
  ↓
Reverse shell
  ↓
Flag location
```

Do not add unrelated exploitation techniques simply because they are interesting.

---

## 🖼️ Screenshot and Asset Guidelines

Screenshots should provide meaningful evidence.

### Good screenshot content

Include evidence such as:

- Service enumeration results.
- HTTP responses.
- Source-code observations.
- Application workflow.
- Upload behavior.
- Safe Zip Slip validation.
- File-write confirmation.
- Hook-processing evidence.
- Sanitized shell access.

### Avoid

- Duplicate screenshots.
- Excessive terminal output.
- Personal information.
- Browser session tokens.
- API keys.
- Unnecessary credentials.
- Unredacted flags.
- Screenshots that do not add technical evidence.

### Image accessibility

Every documentation image should have useful alt text:

```markdown
![Nmap scan showing SSH and HTTP services](assets/02-recon-nmap.svg)
```

Avoid:

```markdown
![screenshot](assets/image.png)
```

---

## 🔐 Security and Sensitive Information

Security documentation must not accidentally expose sensitive information.

Before submitting a contribution, inspect your changes for:

- API keys.
- Passwords.
- Session cookies.
- Authentication tokens.
- SSH private keys.
- Personal information.
- Internal URLs.
- Cloud credentials.
- Real production IP addresses.
- Secrets copied from your local environment.

Use placeholders when an example requires a value:

```text
YOUR_ATTACKER_IP
TARGET_IP
YOUR_PORT
REDACTED
```

Do not commit real secrets simply because they were visible during testing.

---

## 🚩 Flag Redaction Policy

This repository intentionally keeps the final TryHackMe flag **redacted**.

Contributors must not add the actual challenge flag to:

- `README.md`
- `Documentation/`
- `docs/`
- `Resources/`
- Screenshots
- SVG assets
- Commit messages
- Pull-request descriptions
- Example output
- Generated documentation

Use:

```text
[REDACTED]
```

or:

```text
FLAG REDACTED
```

when the flag needs to be referenced.

### Why?

The purpose of this repository is to document:

- Enumeration methodology.
- Vulnerability analysis.
- Exploitation reasoning.
- Evidence.
- Defensive lessons.

It is not intended to become a public answer key containing the challenge flag.

---

## 🧪 Recommended Workflow

Before making changes:

```bash
git clone https://github.com/anurag-rvnkr1/The-Hollow-Shell-TryHackMe-Walkthrough.git
cd The-Hollow-Shell-TryHackMe-Walkthrough
```

Create a working branch:

```bash
git checkout -b docs/improve-walkthrough
```

Make the required changes.

Review the repository:

```bash
git status
git diff
```

Check for accidental secrets or flags before committing.

If GitHub Pages content was modified, verify the Markdown and asset paths locally where practical.

Then commit:

```bash
git add .
git commit -m "docs: improve hollow shell walkthrough"
```

Push the branch:

```bash
git push origin docs/improve-walkthrough
```

Open a pull request with a clear description of the changes.

---

## 💬 Commit Message Guidelines

Use short, descriptive commit messages.

Recommended format:

```text
type: short description
```

Examples:

```text
docs: improve zip slip explanation
docs: redact sensitive evidence
docs: update attack chain diagram
fix: correct broken asset path
fix: repair github pages navigation
style: improve documentation formatting
refactor: reorganize evidence assets
chore: clean unused screenshots
```

### Common commit types

| Type | Purpose |
|---|---|
| `docs` | Documentation changes |
| `fix` | Bug or documentation correction |
| `style` | Formatting or presentation |
| `refactor` | Structural improvement |
| `chore` | Maintenance |
| `test` | Validation or testing changes |

Keep commits focused. Avoid combining unrelated changes into a single commit.

---

## 🔍 Pull Request Guidelines

A good pull request should explain:

### 1. What changed?

Example:

> Improved the Zip Slip section and added a clearer explanation of the extraction boundary.

### 2. Why was it changed?

Example:

> The previous explanation described the payload but did not explain why the traversal escaped the intended directory.

### 3. What was tested?

Example:

> Verified Markdown links, image paths, and GitHub Pages-compatible asset references.

### 4. Are sensitive values redacted?

Confirm that:

- [ ] Flags are redacted.
- [ ] Credentials are not unnecessarily exposed.
- [ ] Tokens and secrets are absent.
- [ ] Personal information is absent.

---

## ✅ Quality Checklist

Before submitting a contribution, verify:

### Technical accuracy

- [ ] Commands are correct.
- [ ] Technical claims are accurate.
- [ ] Vulnerabilities are described correctly.
- [ ] Exploitation steps remain within authorized lab scope.
- [ ] The documented attack chain is internally consistent.

### Documentation

- [ ] Markdown renders correctly.
- [ ] Headings follow a logical hierarchy.
- [ ] Code blocks specify the appropriate language.
- [ ] Links work.
- [ ] Images load correctly.
- [ ] Alt text is descriptive.
- [ ] No unnecessary duplicate content exists.

### Security

- [ ] No real-world credentials are included.
- [ ] No API keys or tokens are included.
- [ ] No private keys are included.
- [ ] No personal information is included.
- [ ] The final flag remains redacted.
- [ ] Payload examples use placeholders where appropriate.

### GitHub Pages

If modifying `docs/`:

- [ ] Relative asset paths are correct.
- [ ] Front matter is valid where required.
- [ ] Navigation remains functional.
- [ ] CSS changes do not break readability.
- [ ] Images render correctly on the deployed site.

---

## 🐛 Reporting Issues

If you find a problem but do not want to modify the repository directly, open an issue describing:

1. The affected file or section.
2. What appears to be incorrect.
3. Why it is incorrect.
4. A proposed correction, if known.
5. Any relevant evidence.

Example:

```text
Title:
Fix incorrect extraction-path explanation

Description:
The Zip Slip section currently describes the extraction directory
incorrectly. The documentation should clarify that the archive entry
can escape the intended destination when path validation is absent.

Suggested correction:
Update the extraction-boundary diagram and accompanying explanation.
```

Do not include the challenge flag in an issue.

---

## 🔒 Security Vulnerability Reporting

If you discover a security issue in the **repository itself** rather than the TryHackMe challenge, avoid publicly posting sensitive exploit details until the issue has been reviewed.

See [`SECURITY.md`](SECURITY.md) for the repository's security-reporting guidance.

Remember that vulnerabilities demonstrated against the TryHackMe target are part of the authorized lab environment and should not be treated as vulnerabilities in unrelated production systems.

---

## 📖 Attribution and References

Contributions should preserve appropriate attribution when using:

- External documentation.
- Security standards.
- Public research.
- Tool documentation.
- Open-source code.
- Third-party assets.

Do not copy substantial text from another walkthrough and present it as original work.

When referencing external material, prefer authoritative sources such as:

- TryHackMe room documentation.
- OWASP.
- CWE.
- Official tool documentation.
- Vendor documentation.
- Original research publications.

---

## 🌐 Documentation Philosophy

This project follows a simple principle:

> **Document the reasoning, not just the commands.**

A strong security write-up should allow a reader to understand:

```text
What was discovered?
        ↓
Why was it interesting?
        ↓
What hypothesis was tested?
        ↓
How was it safely validated?
        ↓
What vulnerability was confirmed?
        ↓
How did the vulnerability enable the next stage?
        ↓
What security lesson can be taken from it?
```

This makes the repository useful as both a **CTF record** and a **professional cybersecurity portfolio project**.

---

## 🙌 Thank You

Every correction, technical clarification, documentation improvement, and accessibility enhancement helps make this project better.

If you contribute, please keep the repository:

**accurate · original · reproducible · secure · professional**

---

<p align="center">
  <strong>🐚 The shell was hollow. The upload boundary wasn't.</strong>
</p>

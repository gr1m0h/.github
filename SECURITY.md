# Security Policy

This policy applies to every repository owned by [@gr1m0h](https://github.com/gr1m0h) that does not ship its own `SECURITY.md`. Individual repos may override or extend this baseline.

## Reporting a vulnerability

**Please do not open a public Issue or Pull Request for security findings.**
The preferred channel is GitHub's [private vulnerability reporting][pvr]:

1. Open the affected repository.
2. Go to **Security → Report a vulnerability**.
3. Fill in the form. Drafts are visible only to you and the maintainer.

If the repository does not have private vulnerability reporting enabled, or
you cannot use GitHub for the report, email **dev@grimoh.net** instead.
Use a subject line that begins with `[SECURITY]` so the report does not get
buried.

[pvr]: https://docs.github.com/code-security/security-advisories/guidance-on-reporting-and-writing/privately-reporting-a-security-vulnerability

### What to include

Reports are most actionable when they contain:

- The affected repository and version / commit SHA
- A short description of the impact (what an attacker can achieve)
- Reproduction steps (proof-of-concept, command line, sample input)
- Your assessment of severity (e.g. CVSS vector) — optional but appreciated
- Whether you intend to publish a write-up, and the timeline you have in mind

You do not need a working exploit. A clear description of a credible attack path is enough to start triage.

## Response timeline

This is a solo-maintained set of projects. Reasonable expectations:

| Stage                       | Target                              |
|-----------------------------|-------------------------------------|
| Acknowledgement of report   | Within 5 business days              |
| Initial assessment          | Within 14 business days             |
| Fix or mitigation in `main` | Best effort; depends on severity    |
| Coordinated public disclosure | Together with the reporter        |

If you have not received an acknowledgement within 7 days, please re-send the report through the alternate channel (PVR ↔ email).

## Scope

### In scope

- Code, configuration, and release artifacts in any [@gr1m0h](https://github.com/gr1m0h)-owned repository
- Documented behaviour of published tools (e.g. CLI output, action inputs)
- Supply-chain integrity of releases (signatures, attestations, lockfiles)

### Out of scope

- Vulnerabilities in third-party dependencies — please report those upstream. If a dependency vulnerability has a viable workaround in our code, that workaround is in scope.
- Findings that require physical access to a maintainer's machine
- Self-XSS, clickjacking on docs sites, and similar findings on non-authenticated static pages
- Volumetric / denial-of-service attacks against public endpoints we do not operate

## Coordinated disclosure

Once a fix is ready, I will:

1. Publish a GitHub Security Advisory (GHSA) on the affected repository.
2. Request a CVE through the GHSA flow when the severity warrants it.
3. Release a patched version and link the advisory.
4. Credit the reporter in the advisory and release notes, unless the reporter prefers to remain anonymous.

Please refrain from publishing details until a patched release is available and the advisory is live, or until 90 days from the initial report — whichever comes first.

## Safe harbour

Good-faith security research on these projects is welcomed and will not be pursued as hostile activity, provided you:

- Make a reasonable effort to avoid privacy violations, data destruction, or
  service interruption
- Do not exfiltrate data beyond what is needed to demonstrate the issue
- Give the maintainer reasonable time to remediate before public disclosure
- Do not exploit the vulnerability beyond proving it exists

## Recognition

Reporters of valid findings are credited in the GHSA and in the release notes of the patched version, unless they prefer not to be named. This project does not currently offer a paid bug bounty.

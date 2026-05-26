---
name: security-audit
description: Run a CSWG-aligned security audit on the current codebase
user-invocable: true
argument-hint: "[<pr-number> | <github-pr-url>] [output-directory]"
---

# CSWG-Aligned Security Audit

> **IMPORTANT — PLAN AND REPORT ONLY.**
> This skill produces an audit report and remediation plan. It does **not** make
> any code changes. Do not edit, create, or delete any source files. All findings
> and recommendations are written to the output report exclusively.

## Invocation Modes

Parse `$ARGUMENTS` to determine the audit mode before doing anything else:

| Invocation | Mode | Output filename |
|---|---|---|
| `/security-audit` | Full local repo audit (current branch) | `YYYY-MM-DD-<project>-<branch>-SECURITY-AUDIT.md` |
| `/security-audit 123` | PR audit — bare PR number | `YYYY-MM-DD-<project>-PR-123-SECURITY-AUDIT.md` |
| `/security-audit https://github.com/…/pull/123` | PR audit — GitHub PR URL | `YYYY-MM-DD-<project>-PR-123-SECURITY-AUDIT.md` |
| Either mode accepts an optional trailing output directory path. If omitted, write to the current working directory. |

---

### Mode A — Full Local Repo Audit

Audit the entire local repository on the current branch. Assume production code.
Skip generated files, vendored dependencies, lock files, and build artifacts.

Derive repository metadata from the conversation context (working directory name,
git status, recent commits) and today's date. Do NOT run git remote commands.
Only use local git commands (e.g. `git rev-parse HEAD`) if the full commit SHA is
not already available in the conversation context.

**Audit scope**: All source files in the repository.

---

### Mode B — PR Audit

Triggered when `$ARGUMENTS` is a bare integer (e.g. `230`) or a GitHub PR URL
(e.g. `https://github.com/<org>/<repo>/pull/230`). Extract the PR number from
whichever form is provided and use it for all subsequent `gh pr` calls.

**Step 1 — Gather PR context:**
```
gh pr view <number> --json number,title,body,author,baseRefName,headRefName,additions,deletions,changedFiles,state
gh pr diff <number>
```

**Step 2 — Read changed files in full:**
For every file reported as changed, read its current full content — not just the
diff lines. Diff context alone is insufficient for tracing injection paths,
understanding auth flows, or detecting secrets.

**Step 3 — Determine audit scope:**
- Primary scope: all code introduced or modified by the PR.
- Extended scope: any function, module, or config that the PR's new code directly
  calls or depends on, if it contains a security-relevant pattern. Clearly label
  extended-scope findings as `[context — not introduced by this PR]`.

**Audit scope note in report**: Include a one-paragraph scope statement below
the header block explaining what the PR changes and what was examined.

---

This audit is structured around what a Cyber Security Working Group (CSWG) actually
assesses — aligned to recognised security standards and frameworks.

## Ground Rules

- Map the platform, framework, concurrency model, entry points, and
  execution flow before starting.
- Trace execution paths before asserting behaviour. State what DOES happen,
  not what "could" or "might" happen.
- Critical/High findings must include a concrete exploit scenario with
  attacker action, precondition, and observable impact. If you cannot
  describe one, lower the severity.
- Identify root causes, not symptoms.
- Analyse cross-file interactions — do not review files in isolation.

## The 12 CSWG Review Areas

### 1. Authentication & Access Control
- AuthN/AuthZ mechanisms, session management, privilege escalation
- Credential storage, password hashing, MFA implementation
- Token lifecycle (issuance, validation, revocation, expiry)
- OAuth 2.0 / OIDC: redirect_uri validation, PKCE enforcement on public clients,
  implicit flow deprecation, authorization code interception, token leakage via
  Referer headers or logs

### 2. Data Protection & Privacy
- Encryption at rest and in transit, PII handling, data classification
- Retention and disposal policies, GDPR/CCPA compliance
- Data minimisation, consent tracking, right to deletion

### 3. Input Validation & Injection Prevention
- SQL, XSS, command, LDAP, template injection vectors
- Input sanitisation, parameterised queries, output encoding
- Content-Type validation, file upload restrictions
- SSRF (Server-Side Request Forgery, OWASP A10:2021 / API7:2023) — unvalidated
  URLs passed to internal HTTP clients; check for access to private IP ranges
  and cloud metadata endpoints (169.254.169.254, fd00:ec2::254)
- XXE (XML External Entity injection, CWE-611) — verify external entity
  resolution is disabled in XML parsers (libxml2, .NET XmlDocument,
  Java DocumentBuilder)
- TOCTOU race conditions (CWE-362) — time-of-check/time-of-use flaws in
  concurrent or async code affecting file access, authentication state, or
  shared resources

### 4. Secrets Management
- Hardcoded credentials, API keys, tokens in source code
- `.gitignore` audit — flag sensitive files not excluded (`.env`, creds,
  keystores, private keys)
- Git history scan for committed secrets, secret rotation practices
  - Recommended tools: trufflehog, gitleaks (git history scanning);
    detect-secrets, git-secrets (pre-commit prevention)
- Vault/secrets manager usage

### 5. Dependency & Supply Chain Security
- Known CVEs — run `npm audit`, `dotnet list package --vulnerable`,
  `pip audit`, or equivalent for the project's ecosystem
- Dependency pinning, integrity verification (lock file hashes)
- Vendored code audit, SCA coverage
  - Container/IaC scanning: trivy, grype
  - Multi-ecosystem SCA: snyk, OWASP Dependency-Check
  - SAST: semgrep, bandit (Python), gosec (Go), eslint-plugin-security (JS/TS)
- Unsafe consumption of third-party APIs (OWASP API10:2023) — unvalidated data
  from external API responses used in business logic or rendered to users

### 6. Secure Communications
- TLS configuration, certificate validation, protocol versions
- Certificate pinning, API transport security
- Internal service-to-service communication security

### 7. Logging, Monitoring & Audit Trail
- Security event logging (auth attempts, privilege changes, data access)
- PII in logs, log integrity, tamper resistance
- SIEM readiness, alerting on security-relevant events

### 8. Error Handling & Information Disclosure
- Stack traces in production, debug modes exposed
- Verbose error messages leaking internals to users
- Exception handling that reveals system architecture

### 9. Security Configuration & Hardening
- Default credentials, CORS policy, CSP and security headers
- Unnecessary services, open ports, debug endpoints
- Environment-specific security settings (dev vs prod)
- IaC / Container security:
  - Dockerfile: USER directive (non-root), privileged flags, secrets in image
    layers or ENV instructions
  - Kubernetes: RBAC over-permission, NetworkPolicy gaps, pod security
    standards, exposed LoadBalancer services
  - Terraform/CloudFormation: over-permissive IAM policies, public S3 buckets,
    unencrypted storage, open security groups (0.0.0.0/0)
- If the codebase targets Android or iOS: assess against OWASP MASVS v2.0 and
  OWASP Mobile Top 10 2024 — insecure data storage (M2: SharedPreferences,
  SQLite, external storage); improper platform usage (M1: exported components,
  deep links, pending intents, URL schemes); insufficient binary protections
  (M9: obfuscation, certificate pinning where risk warrants it, root/jailbreak
  detection for high-risk apps); WebView misconfigurations (M7:
  addJavascriptInterface, file:// access, mixed content); exported component
  attack surface (AndroidManifest android:exported, intent filters).

### 10. Cryptography
- Algorithm strength and key lengths
- Deprecated algorithms (MD5, SHA1, DES, RC4)
- RNG quality (use of cryptographically secure sources)
- Key management lifecycle

### 11. Business Logic & Authorisation
- IDOR vulnerabilities, mass assignment
- Broken Object Property Level Authorization (OWASP API3:2023) — responses
  returning more fields than the caller is entitled to see
- Rate limiting, abuse prevention; unrestricted resource consumption /
  DoS exhaustion (OWASP API4:2023)
- Privilege boundary enforcement, transaction integrity
- Workflow bypass, state manipulation

### 12. Secure SDLC & Compliance Posture
- SAST/DAST integration in CI/CD pipeline
- Security testing evidence, code review practices
- Regulatory alignment, code signing, deployment security
- Branch protection, merge requirements
- Shadow and deprecated API endpoint inventory (OWASP API9:2023) —
  undocumented or retired endpoints still reachable in production
- Supply chain integrity:
  - SLSA framework alignment — build provenance, hermetic builds, verified
    artifact integrity
  - Dependency confusion / typosquatting risk in npm/PyPI/NuGet namespaces
  - CI/CD pipeline integrity — pinned action versions (GitHub Actions SHA
    pinning), no untrusted third-party runners
  - Software Bill of Materials (SBOM) presence and completeness (SPDX or
    CycloneDX format)
- If the codebase integrates LLM/AI features: assess against OWASP LLM
  Top 10 2025 — prompt injection (LLM01), insecure output handling (LLM02),
  excessive agency (LLM08)

## Severity Definitions

- **Critical**: Actively exploitable. Data breach, auth bypass, RCE.
  Concrete exploit scenario required.
- **High**: Exploitable under realistic conditions. Requires concrete example.
- **Medium**: Security weakness, not immediately exploitable.
  Defence-in-depth gap.
- **Low**: Hardening recommendation. Best practice deviation.

If using "could", "might", or "potentially" — that is Medium or below.

## Grading Rubric

Derive the Overall Security Grading score from finding counts:

| Score | Criteria |
|-------|----------|
| 9-10  | No Critical/High. <= 2 Medium. CSWG ready. |
| 7-8   | No Critical. <= 1 High, <= 4 Medium. CSWG ready with minor remediation. |
| 5-6   | No Critical. 2+ High or 5+ Medium. Needs remediation before CSWG. |
| 3-4   | 1+ Critical or 4+ High. Significant security risk. Not CSWG ready. |
| 1-2   | Multiple Critical. Fundamental security failures. Not CSWG ready. |

Modifiers (each applies independently, cumulative):
- -1 if findings cluster in Authentication, Secrets, or Cryptography
  (high CSWG scrutiny areas)
- -1 if 10+ Low findings (indicates systemic hardening debt)
- -1 if this is a re-audit and zero findings from the previous audit have been
  remediated (indicates stalled remediation posture)

Final score is clamped to 1-10.

## Finding Format

```
### [Short Title]
**Severity**: Critical / High / Medium / Low
**CWE**: CWE-XXX (name) | **OWASP**: A0X:YYYY (category)
**CVSS**: <base score> (AV:_/AC:_/PR:_/UI:_/S:_/C:_/I:_/A:_)  [required for Critical/High; omit for Medium/Low]
**Location**: file:line
**What happens**: [Actual runtime behaviour — traced, not speculated]
**Exploit scenario**: [attacker action -> precondition -> observable impact]
  (required for Critical/High; optional for Medium/Low)
**Root cause**: [Underlying reason, if different from symptom]
**Recommendation**: [Specific fix or mitigation]
**Status**: Open / Remediated / False Positive — <rationale>  [optional, for re-audit tracking]
```

## Output Structure

1. **Header** — metadata block in this exact format:

```
Repository Name     : <from folder name or git log — no git remote commands>
Branch Name / Commit: <current branch> (<full commit SHA>)
Project             : <from README or package manifest>
Technology Stack    : <primary language, framework, version>
Analysis Date       : <YYYY-MM-DD>
Reviewer            : <from git config user.name or conversation context>
Report Version      : 1.0
Audit Mode          : Full Repo / PR #<number> — <PR title>
Audit Scope         : <Full codebase on <branch> / PR #N: <N files changed, +X −Y lines>>
Overall Security Grading: <X/10 — derived from findings>
CSWG Ready          : <Ready / Ready with caveats / Not ready — one-line justification>
Frameworks Referenced: OWASP ASVS v5.0.0, OWASP Top 10 2021, OWASP API Security Top 10 2023,
                       CWE Top 25, NCSC CAF, NIST CSF, ISO 27001 Annex A, OWASP LLM Top 10 2025,
                       OWASP MASVS v2.0, OWASP Mobile Top 10 2024 (if mobile)
```

Source values from repo metadata. Write "Unknown" if undetermined.

For PR audits: follow the header with a **Scope Statement** paragraph — one
short paragraph summarising what the PR does and which files were examined.
This orients the reader before findings begin.

2. **Management Summary** — self-contained section for non-technical stakeholders:
   - Finding count table:

     | Severity | Count |
     |----------|-------|
     | Critical | N     |
     | High     | N     |
     | Medium   | N     |
     | Low      | N     |

   - 2-3 sentences on overall security posture in plain language
   - Top 5 risks ranked by business impact — one sentence each, no code
     references, no line numbers
   - CSWG readiness verdict: Ready / Ready with caveats / Not ready — with
     one-line justification
   - Estimated remediation effort per priority tier:
     S (hours) / M (days) / L (sprint+)
   - A manager should be able to read only this section and understand the
     security state of the codebase

3. **Threat Model Overview** — entry points, trust boundaries, data flows,
   attack surface. Use a plain-text ASCII box-and-arrow diagram. No Mermaid.

   Apply STRIDE per-element: for each component in the diagram, enumerate
   applicable threats (Spoofing, Tampering, Repudiation, Information Disclosure,
   Denial of Service, Elevation of Privilege). Identify external actors, entry
   points, trust boundaries crossed, and sensitive data stores.

4. **Findings** — grouped by the 12 CSWG review areas, sorted by severity
   within each group. Every review area must appear — either with findings,
   an explicit "no issues found" statement, or (PR mode only) "not touched
   by this PR" where the changed files have no relevance to the area.

5. **CSWG Readiness Assessment** — pass/fail table covering each of the
   12 areas:

```
| # | Area | Status | Key Finding | Remediation Required |
```

Status values: PASS / FAIL / PARTIAL — with one-line justification.

6. **Remediation Plan** — prioritised by CSWG impact:
   - P0: Must fix before CSWG submission (Critical/High)
   - P1: Should fix, will likely be flagged (Medium with high visibility)
   - P2: Improve when possible (remaining Medium/Low)
   - Use JIRA-ready action titles (imperative, concise)

   **Retest Guidance**: For each P0/P1 finding, document the verification steps
   and expected evidence confirming remediation (e.g. unit test output, config
   diff, tool scan result, log entry).

## Quality Calibration

State facts, not speculation:

- NO: "Potential XSS vulnerability"
  YES: "User input from `req.query.search` at line 42 is rendered via
  `innerHTML` at line 68 without sanitisation, allowing script injection
  via the search endpoint"

- NO: "Secrets may be exposed"
  YES: "AWS access key `AKIA...` committed in `config/aws.js:12`, rotated
  0 times based on git history, granting S3 read/write to production bucket"

- NO: "Weak cryptography used"
  YES: "Password hashing in `auth/hash.js:15` uses MD5 with no salt,
  allowing rainbow table attacks against the user credential store"

## Completeness Check

Before finalising the report, verify:

- [ ] Management Summary is self-contained and jargon-free
- [ ] Audit mode and scope are clearly stated in the header block
- [ ] PR audit: Scope Statement paragraph present below the header
- [ ] Every review area (1-12) is addressed — findings, "no issues found",
  or (PR mode) "not touched by this PR"
- [ ] All Critical/High findings include an exploit scenario
- [ ] All Critical/High findings include a CVSS base score and vector
- [ ] Overall Security Grading matches the Grading Rubric criteria (all
  three modifiers checked, including re-audit regression where applicable)
- [ ] If Android/iOS project: OWASP MASVS / Mobile Top 10 coverage noted in
  Area 9 findings or explicitly dismissed as N/A
- [ ] CSWG Ready assessment is consistent with the grading
- [ ] CSWG Readiness Assessment table is complete (all 12 areas)
- [ ] Threat model diagram included with STRIDE per-element analysis
- [ ] Header metadata is fully populated (no placeholder values remaining)
- [ ] Remediation plan covers all Critical/High findings
- [ ] Retest Guidance included for all P0/P1 findings

## Done When

- [ ] `.md` file written to the output directory with the correct filename
- [ ] No source files were edited, created, or deleted
- [ ] No remediation steps were applied to the codebase

---

## Appendix: NCSC CAF Objective Mapping

The NCSC Cyber Assessment Framework (CAF) is referenced in this skill's
framework list. Its four top-level objectives map to the 12 review areas
as follows:

| CAF Objective | Description | Review Areas |
|---|---|---|
| A — Managing Security Risk | Governance, risk management, asset management | 1, 4, 10, 12 |
| B — Protecting Against Cyber Attack | Secure configuration, identity/access, data security, system protection | 1, 2, 3, 5, 6, 9, 10, 11 |
| C — Detecting Cyber Security Events | Security monitoring, anomaly detection | 7 |
| D — Minimising the Impact of Incidents | Response, recovery, lessons learned | 7, 8, 12 |

---

## Revision History

| Version | Date       | Changes |
|---------|------------|---------|
| 1.4     | 2026-05-26 | Removed disable-model-invocation frontmatter flag (prose guard is sufficient; flag had ambiguous harness semantics). Made CVSS base score required for Critical/High findings, explicitly omitted for Medium/Low. Added re-audit regression modifier (-1 if zero prior findings remediated). Added conditional OWASP MASVS v2.0 / Mobile Top 10 2024 coverage to Area 9 for Android/iOS projects. Added OWASP MASVS v2.0 and Mobile Top 10 2024 (if mobile) to Frameworks Referenced header template. Updated Completeness Check with CVSS and mobile coverage checklist items. Updated grading rubric note to reference all three modifiers. |
| 1.3     | 2026-05-26 | PR mode argument syntax updated: bare PR number (`123`) and GitHub PR URL (`https://github.com/.../pull/123`) now accepted directly — the `pr` prefix is no longer required. |
| 1.2     | 2026-05-26 | Added two invocation modes: Mode A (full local repo audit, default) and Mode B (PR audit via `pr <number>`). PR mode fetches PR metadata and diff via `gh pr`, reads changed files in full, scopes findings to the PR with clearly labelled extended-scope context findings, and adds a Scope Statement below the header. Header block now includes Audit Mode and Audit Scope fields. Completeness check updated for PR mode. |
| 1.1     | 2026-05-26 | Updated OWASP ASVS reference to v5.0.0 (released May 2025, major version). Added OWASP API Security Top 10 2023 to frameworks and coverage across Areas 3, 5, 11, 12 (API3/BOPLA, API4/resource exhaustion, API7/SSRF, API9/shadow APIs, API10/unsafe API consumption). Added SSRF, XXE/CWE-611, TOCTOU/CWE-362 to Area 3. Expanded OAuth 2.0/OIDC attack vectors in Area 1. Added IaC and container security to Area 9. Added supply chain security (SLSA, SBOM, CI/CD pinning, dependency confusion) to Area 12. Named specific secret scanning tools (trufflehog, gitleaks) in Area 4. Named SAST and container/SCA tools (semgrep, trivy, snyk, etc.) in Area 5. Added CVSS base score and Status fields to finding format. Added Low-volume grading modifier (-1 if 10+ Lows). Added Retest Guidance to Remediation Plan section. Added STRIDE per-element methodology to Threat Model section. Added conditional LLM/AI coverage (OWASP LLM Top 10 2025) to Area 12. Fixed git remote contradiction in header template. Added Report Version field and full Low count to Management Summary. Added NCSC CAF objective mapping appendix. Added OWASP LLM Top 10 2025 to Frameworks Referenced. |
| 1.0     | (original) | Initial release — 12 CSWG review areas, severity definitions, grading rubric, finding format, output structure, completeness checklist. |

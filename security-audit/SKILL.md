---
name: security-audit
description: Run a CSWG-aligned security audit (full repo or PR diff)
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

In either mode, an optional trailing output directory path may be appended. If omitted, write to the current working directory.

Sanitise the `<project>` and `<branch>` filename components: lowercase, replace spaces and slashes with hyphens, strip non-alphanumeric characters (excluding hyphens).

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

If the PR changes more than 20 files, prioritise by risk: authentication/session
files, cryptographic modules, input validation handlers, configuration files, and
any file with `admin`, `token`, `key`, `secret`, or `password` in its path.
Examine lower-priority files via diff only. Note in the Scope Statement which
files were reviewed in full vs. diff-only and why.

**Step 3 — Determine audit scope:**
- Primary scope: all code introduced or modified by the PR.
- Extended scope: packages that are newly added by this PR, or whose version this
  PR explicitly changed (e.g. a semver bump in `package.json`, `requirements.txt`,
  `go.mod`). Run the ecosystem's audit tool (`npm audit`, `pip audit`, etc.) and
  surface CVEs only for those packages. Clearly label these findings as
  `[context — not introduced by this PR]`.
- Out of scope: pre-existing dependencies not touched by this PR, existing source
  files not modified by this PR, and general baseline codebase issues. These belong
  in a full-repo audit (Mode A), not a PR audit.

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

## Audience Calibration

This skill produces **developer-oriented security reviews**, not security consulting firm engagements.

- Reports must be achievable: prioritise findings that a development team can address within a normal sprint
- Aim for 10–20 actionable findings per report; avoid padding with theoretical risks that have no practical exploit path in the actual deployment context
- Severity reflects real-world exploitability in the codebase's actual deployment context, not CVSS theoretical maximums
- Critical = exploitable with realistic effort in this deployment. High = fix this sprint. Medium = backlog within 30 days. Low = address when touching the relevant code.

## The 12 CSWG Review Areas

### 1. Authentication & Access Control
- AuthN/AuthZ mechanisms, session management, privilege escalation
- Credential storage, password hashing, MFA implementation
- Token lifecycle (issuance, validation, revocation, expiry); JWT-specific:
  verify alg:none rejection, algorithm confusion resistance (RS256/HS256
  key-confusion attack, CWE-327), and that jku/kid header values are validated
  against a fixed allowlist and not resolved as attacker-controlled URLs (CWE-347)
- OAuth 2.0 / OIDC: redirect_uri validation, PKCE enforcement on public clients,
  implicit flow deprecation, authorization code interception, token leakage via
  Referer headers or logs
- OAuth 2.1 (current standard — formerly-optional controls are now mandatory):
  PKCE required for **all** clients (not just public clients); redirect URI must be
  exact string match (no wildcards or partial patterns); implicit grant flow is
  **removed** — any usage is a High finding; Resource Owner Password Credentials
  flow removed; bearer tokens prohibited in URL query strings
- WebAuthn / Passkeys (if applicable): verify account recovery flows (SMS/email)
  cannot bypass the passkey step — recovery is the weakest link; verify XSS
  cannot trigger attacker-controlled passkey registration (attacker can add their
  key even without stealing the user's private key); for high-assurance
  applications, flag synced passkeys (cross-device cloud sync) as weaker than
  device-bound credentials
- CSRF (Cross-Site Request Forgery, CWE-352, OWASP A01:2025) — verify CSRF
  tokens or SameSite=Strict/Lax cookie attributes on all state-changing
  requests; check HTML forms, API mutation endpoints, and single-page app
  fetch calls for absent anti-CSRF controls

### 2. Data Protection & Privacy
- Encryption at rest and in transit, PII handling, data classification
- Retention and disposal policies, GDPR/CCPA compliance
- Data minimisation, consent tracking, right to deletion
- Cleartext storage of sensitive data (CWE-312) — PII, credentials, or tokens written
  unencrypted to disk, databases, or logs
- Cleartext transmission of sensitive data (CWE-319) — PII or credentials in unencrypted
  channels; check internal service calls, not just external-facing APIs
- Data masking and tokenisation — PII fields (SSN, card numbers, health data) masked or
  tokenised before storage; check ORM serialisers and API response schemas for over-exposure
  of sensitive fields

### 3. Input Validation & Injection Prevention
- SQL, XSS, command, LDAP, template injection vectors
- Input sanitisation, parameterised queries, output encoding
- Content-Type validation, file upload restrictions
- SSRF (Server-Side Request Forgery, CWE-918 / API7:2023) — unvalidated
  URLs passed to internal HTTP clients; check for access to private IP ranges
  and cloud metadata endpoints (169.254.169.254, fd00:ec2::254).
  [Note: SSRF was A10:2021; it has no standalone position in OWASP Top 10 2025
  and is commonly classified under A01:2025 Broken Access Control]
- XXE (XML External Entity injection, CWE-611) — verify external entity
  resolution is disabled in XML parsers (libxml2, .NET XmlDocument,
  Java DocumentBuilder)
- TOCTOU race conditions (CWE-362) — time-of-check/time-of-use flaws in
  concurrent or async code affecting file access, authentication state, or
  shared resources
- Path traversal (CWE-22) — file path inputs not canonicalised before use;
  check for `../` sequences in upload destinations, download handlers,
  template loaders, log file paths, and static file servers
- Open redirect (CWE-601) — `redirect`, `next`, `return_to`, and `url`
  parameters passed to HTTP 302 responses without allowlist validation;
  enabler for OAuth redirect_uri phishing
- Insecure deserialization (CWE-502, OWASP A08:2025) — untrusted serialised
  objects processed without integrity validation; Java ObjectInputStream,
  Python pickle, PHP unserialize, and .NET BinaryFormatter are high-risk
  when fed attacker-controlled input; can lead to RCE
- If the codebase is written in C, C++, or another memory-unsafe language:
  assess for memory safety vulnerabilities — out-of-bounds writes (CWE-787,
  CWE Top 25 #2, 18 KEV CVEs), out-of-bounds reads (CWE-125), use-after-free
  (CWE-416), buffer overflows (CWE-119), and integer overflows feeding size
  calculations (CWE-190). Findings in this class with remote exploitability
  are treated as Critical.
- HTTP request smuggling (CWE-444) — conflicting Content-Length/Transfer-Encoding
  headers exploited when a front-end proxy and back-end server disagree on request
  boundaries; check server pairings (nginx+gunicorn, HAProxy+Express, CDN+origin)
  for desync risk
- DOM-based XSS (CWE-79) — sources: `location.hash`, `document.referrer`,
  `postMessage` data, URL params; sinks: `innerHTML`, `document.write`, `eval`,
  `setTimeout(string)`; requires client-side JS analysis separate from server-side
  template rendering
- Prototype pollution (CWE-1321) — in JavaScript/Node.js codebases: attacker-controlled
  keys (`__proto__`, `constructor.prototype`) merged into objects via `_.merge`,
  `Object.assign`, or `JSON.parse` on untrusted input; can escalate to RCE via
  gadget chains
- ReDoS (CWE-1333) — user-controlled input matched against catastrophically backtracking
  regexes (e.g. `(a+)+` patterns); verify validation regexes are linear-time or use
  timeout/length guards on untrusted input
- If the codebase is a **Ruby on Rails** application:
  - Mass assignment — verify strong parameters (`params.require(:model).permit(...)`) on
    every controller action; un-scoped `params` or `update(params)` allows attacker-set fields
  - XSS via `html_safe` / `raw` — mark all usages and verify no user-controlled string is
    marked safe without sanitisation
  - ActiveRecord SQL injection — flag `.where("col = '#{value}'")`; use placeholder syntax
    `.where("col = ?", value)` or hash conditions
  - RCE via `send()` with user input — arbitrary method call; flag all `send(params[...])` patterns
  - RCE via `constantize` / `safe_constantize` with user input — class name injection
  - `eval` / `instance_eval` / `class_eval` with user-controlled strings
  - Secrets — verify `config/secrets.yml` and `config/credentials.yml.enc` contain no
    plaintext secrets committed to version control; use `rails credentials:edit`
  - SAST: supplement semgrep with **Brakeman** (Rails-specific static analyser)
- If the codebase is a **Node.js / Express** application:
  - Command injection — `child_process.exec(cmd + userInput)` is High; use `execFile()` or
    `spawn()` with an argument array instead
  - RCE — `eval()` / `new Function(userInput)` with user-controlled strings
  - Path traversal via `require()` — `require(userInput)` loads arbitrary modules; validate paths
  - Template injection — EJS `<%- userInput %>`, Handlebars `{{{ userInput }}}`, Pug
    unescaped interpolation allow script injection
  - Open redirect — `res.redirect(req.query.next)` without allowlist validation
  - Missing security headers — verify `helmet()` middleware is applied to the Express app
    (sets CSP, HSTS, X-Frame-Options, X-Content-Type-Options); absence is a Medium finding
  - Unhandled promise rejections — `process.on('unhandledRejection')` should terminate
    gracefully; swallowed async errors in auth/authz paths can create bypass conditions
  - `npm audit` — run as part of dependency review (see Area 5)

### 4. Secrets Management
- Hardcoded credentials, API keys, tokens in source code
- `.gitignore` audit — flag sensitive files not excluded (`.env`, creds,
  keystores, private keys)
- Git history scan for committed secrets, secret rotation practices
  - Recommended tools: trufflehog, gitleaks (git history scanning);
    detect-secrets, git-secrets (pre-commit prevention)
- Vault/secrets manager usage

### 5. Dependency & Supply Chain Security
> Note: Supply chain security is now OWASP A03:2025 (#3). Treat dependency CVE
> findings and build integrity failures at the same priority as injection and
> access control findings.
- Known CVEs — run `npm audit`, `dotnet list package --vulnerable`,
  `pip audit`, or equivalent for the project's ecosystem
- Dependency pinning, integrity verification (lock file hashes)
- Vendored code audit, SCA coverage
  - Container/IaC scanning: trivy, grype
  - Multi-ecosystem SCA: snyk, OWASP Dependency-Check
  - SAST: semgrep, bandit (Python), gosec (Go), eslint-plugin-security (JS/TS),
    Brakeman (Ruby on Rails)
  - **SAST blind spots**: tools miss approximately half of vulnerabilities — particularly
    business logic flaws, multi-step authorization chains, and data flows through
    third-party library internals. A clean SAST result does not substitute for manual
    review of authentication flows, authorization logic, and high-risk data paths.
- Unsafe consumption of third-party APIs (OWASP API10:2023) — unvalidated data
  from external API responses used in business logic or rendered to users

### 6. Secure Communications
- TLS configuration, certificate validation, protocol versions
- Certificate pinning, API transport security
- Internal service-to-service communication security
- TLS version enforcement — reject TLS 1.0 and 1.1; prefer TLS 1.3; verify server
  configuration explicitly disables deprecated cipher suites
- Improper certificate validation (CWE-295) — disabled hostname verification or
  `InsecureSkipVerify` equivalents in HTTP clients, gRPC stubs, or message brokers
- WebSocket origin validation — `Origin` header not checked on WebSocket upgrade
  requests; verify server-side origin allowlist is enforced
- mTLS for internal service mesh — verify service-to-service calls in zero-trust
  architectures use mutual TLS rather than relying solely on network segmentation

### 7. Logging, Monitoring & Audit Trail
- Security event logging (auth attempts, privilege changes, data access)
- PII in logs, log integrity, tamper resistance
- SIEM readiness, alerting on security-relevant events
- Log injection (CWE-117) — newline characters in user-controlled input embedded into
  log streams allow log record forgery; verify inputs are sanitised or structured
  logging is used with parameterised fields rather than string concatenation

### 8. Error Handling & Information Disclosure (OWASP A10:2025)
- Failing open (CWE-636, OWASP A10:2025) — exceptions in authentication or
  authorisation gates that result in access being granted rather than denied;
  trace try/catch blocks around auth checks to verify deny-by-default on error
- Sensitive data in error messages (CWE-209) — stack traces, DB schemas,
  internal hostnames, or credential fragments in API error responses
- Stack traces in production, debug modes exposed
- Verbose error messages leaking internals to users
- Exception handling that reveals system architecture

### 9. Security Configuration & Hardening
- Default credentials, CORS policy
- Security response headers: HSTS (Strict-Transport-Security, subdomains +
  preload), CSP (Content-Security-Policy), X-Frame-Options or CSP
  frame-ancestors, X-Content-Type-Options: nosniff, Referrer-Policy,
  Permissions-Policy, and COEP/CORP for cross-origin isolation
- Unnecessary services, open ports, debug endpoints
- Environment-specific security settings (dev vs prod)
- IaC / Container security:
  - Dockerfile: USER directive (non-root), privileged flags, secrets in image
    layers or ENV instructions
  - Kubernetes: RBAC over-permission, NetworkPolicy gaps, pod security
    standards, exposed LoadBalancer services
  - Terraform/CloudFormation: over-permissive IAM policies, public S3 buckets,
    unencrypted storage, open security groups (0.0.0.0/0)
- Web cache poisoning — unkeyed HTTP headers (`X-Forwarded-Host`, `X-Original-URL`,
  `X-Forwarded-Proto`) reflected into cached responses; verify CDN/proxy cache key
  configuration excludes attacker-injectable headers
- If the codebase targets Android or iOS: assess against OWASP MASVS v2.0 and
  OWASP Mobile Top 10 2024 — insecure data storage (M9: SharedPreferences,
  SQLite, external storage); improper platform usage (M1: exported components,
  deep links, pending intents, URL schemes); insufficient binary protections
  (M7: obfuscation, certificate pinning where risk warrants it, root/jailbreak
  detection for high-risk apps); WebView misconfigurations (M7:
  addJavascriptInterface, file:// access, mixed content); exported component
  attack surface (AndroidManifest android:exported, intent filters);
  inadequate supply chain security (M2: third-party SDKs without integrity
  verification, AAR/JAR dependencies from untrusted or unpinned sources);
  inadequate privacy controls (M6: excessive permissions, data collection
  beyond stated purpose, missing data minimisation, sensitive data retained
  longer than required).

### 10. Cryptography
- Algorithm strength and key lengths
- Deprecated algorithms (MD5, SHA1, DES, RC4)
- RNG quality (use of cryptographically secure sources)
- Key management lifecycle
- Timing attacks (CWE-208) — non-constant-time equality checks for HMAC signatures,
  API tokens, or session IDs allow statistical recovery via response timing; verify
  use of `hmac.compare_digest()` (Python), `crypto.timingSafeEqual()` (Node.js),
  or equivalent constant-time comparison functions
- KDF recommendations — password hashing must use memory-hard KDFs: Argon2id (preferred),
  bcrypt (cost ≥ 12), or scrypt; PBKDF2 acceptable only with ≥ 600,000 iterations
  (NIST SP 800-132); never use raw SHA-family hashes for passwords
- Nonce/IV reuse (CWE-330) — GCM nonce reuse with the same key is catastrophic
  (breaks both confidentiality and authentication); verify nonce generation uses a
  CSPRNG and never repeats; check for CBC IV reuse and ECB mode usage
- Padding oracle (CBC mode) — verify CBC-mode decryption does not leak padding
  validity through timing or error differences; prefer authenticated encryption (AES-GCM)

### 11. Business Logic & Authorisation
- IDOR (Insecure Direct Object References, CWE-639, OWASP A01:2025) —
  object identifiers in requests not validated against the calling user's
  entitlements; mass assignment (unexposed fields set via bulk-update endpoints)
- Broken Object Property Level Authorization (OWASP API3:2023) — responses
  returning more fields than the caller is entitled to see
- Rate limiting, abuse prevention; unrestricted resource consumption /
  DoS exhaustion (OWASP API4:2023)
- Missing authorization (CWE-862, CWE Top 25 2025 #5) — functions or endpoints that
  simply do not check authorization before performing sensitive operations; in Rails:
  missing `before_action :authenticate_user!` or `authorize!`; in Node/Express:
  middleware that can be bypassed via header manipulation or route ordering
- Privilege boundary enforcement, transaction integrity
- Workflow bypass, state manipulation
- Unrestricted access to sensitive business flows (API6:2023) — verify high-value
  flows (account creation, payment initiation, coupon/referral use, bulk data export)
  have bot detection, CAPTCHA, or anomaly controls preventing automated abuse;
  distinct from rate limiting (API4) — this is about abusing legitimate endpoints
- GraphQL (if applicable) — introspection enabled in production leaks full schema;
  unbounded query depth and complexity without cost analysis enables DoS; batching
  attacks and alias flooding bypass rate limiting; field suggestions leak schema to
  unauthenticated users

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
    pinning), no untrusted third-party runners; `GITHUB_TOKEN` over-permission
    (write-all vs. minimal permissions); `pull_request_target` misuse granting
    repository secrets to fork PR workflows; environment secrets vs. repository
    secrets separation
  - Software Bill of Materials (SBOM) presence and completeness (SPDX or
    CycloneDX format)
- If the codebase integrates LLM/AI features: assess against OWASP LLM Top 10 v2.0 (2025):
  - LLM01 — Prompt Injection: attacker manipulates LLM inputs to override instructions or exfiltrate data
  - LLM03 — Supply Chain Vulnerabilities: compromised model weights, poisoned fine-tuning data, malicious plugins
  - LLM04 — Data and Model Poisoning: backdoor triggers in training data or RAG knowledge base poisoning
  - LLM05 — Improper Output Handling: LLM-generated content used in downstream operations (SQL, shell, HTML) without sanitisation
  - LLM06 — Excessive Agency: LLM granted more permissions or autonomy than required; can trigger unauthorised actions
  - LLM07 — System Prompt Leakage: system prompts containing credentials, sensitive logic, or internal instructions exposed via extraction
  - LLM08 — Vector and Embedding Weaknesses: RAG database poisoning, cross-context data leakage through embeddings

## Severity Definitions

- **Critical**: Actively exploitable. Data breach, auth bypass, RCE.
  Concrete exploit scenario required.
- **High**: Exploitable under realistic conditions. Requires concrete example.
- **Medium**: Security weakness, not immediately exploitable.
  Defence-in-depth gap.
- **Low**: Hardening recommendation. Best practice deviation.

If using "could", "might", or "potentially" — that is Medium or below.

**Chained findings**: When two or more findings combine to produce a higher-severity outcome (e.g. open redirect + CSRF enabling auth bypass), document a separate combined finding at the elevated severity with an explicit chain-of-exploitation scenario referencing the constituent findings by title.

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
  remediated (indicates stalled remediation posture). To detect: if a prior audit
  report exists in the output directory or is referenced in `$ARGUMENTS`, compare
  finding titles; if any previously reported Critical/High findings remain
  unchanged, apply this modifier.

Final score is clamped to 1-10.

## Finding Format

```
### [Short Title]
**Severity**: Critical / High / Medium / Low
**CWE**: CWE-XXX (name) | **OWASP**: A0X:YYYY (category)
**CVSS**: <base score> (CVSS:3.1/AV:_/AC:_/PR:_/UI:_/S:_/C:_/I:_/A:_)  [required for Critical/High; omit for Medium/Low]
         (CVSS v4.0 may be used where the organisation standard requires it —
         format: CVSS:4.0/AV:_/AC:_/AT:_/PR:_/UI:_/VC:_/VI:_/VA:_/SC:_/SI:_/SA:_)
**Location**: file:line
**Scope**: In-scope (introduced by this PR) | Context (pre-existing — not introduced by this PR)  [PR mode only; omit for full-repo audits]
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
Frameworks Referenced: OWASP ASVS v5.0.0 (coverage aligned in spirit; not a verbatim chapter mapping),
                       OWASP Top 10 2025, OWASP API Security Top 10 2023,
                       CWE Top 25 2025, NCSC CAF v4.0, NIST CSF, ISO/IEC 27001:2022 Annex A,
                       OWASP LLM Top 10 v2.0 (2025), OWASP MASVS v2.0, OWASP Mobile Top 10 2024 (if mobile)
```

Source values from repo metadata. Write "Unknown" if undetermined.

For PR audits: follow the header with a **Scope Statement** paragraph — one
short paragraph summarising what the PR does and which files were examined.
This orients the reader before findings begin.

2. **Audit Target** — a brief orientation section inserted immediately after the header
   block. Populate from README, package manifests, git context, and `$ARGUMENTS`.
   Use "Unknown — findings assume internet-accessible deployment" for any unknown field.

```
## Audit Target

**Audience**: Engineering team — experienced developers with security awareness.
  This report is not a security consulting engagement. Findings are calibrated
  to be actionable within a normal development sprint.

**Technology Stack**: <primary language/framework — e.g. Node.js/Express, Ruby on Rails 7, Android Kotlin>

**Deployment Context**: <how this app is deployed — e.g. "Rails API behind nginx on AWS ECS;
  Android client on Play Store">

**Audit Focus**: <what this audit prioritises — e.g. "API surface, authentication flows,
  dependency supply chain, Android data storage">

**Out of Scope**: <what was intentionally excluded — e.g. "infrastructure hardening,
  penetration testing of deployed endpoints">

**Findings Calibration**: Critical = exploitable with realistic effort in this deployment.
  High = fix this sprint. Medium = backlog within 30 days.
  Low = address when touching the relevant code.
```

3. **Management Summary** — self-contained section for non-technical stakeholders:
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

4. **Threat Model Overview** — entry points, trust boundaries, data flows,
   attack surface. Use a plain-text ASCII box-and-arrow diagram. No Mermaid.

   Apply STRIDE per-element: for each component in the diagram, enumerate
   applicable threats (Spoofing, Tampering, Repudiation, Information Disclosure,
   Denial of Service, Elevation of Privilege). Identify external actors, entry
   points, trust boundaries crossed, and sensitive data stores.

   Verify the diagram reflects **actual deployed state**, not design intent —
   discrepancies (undocumented reverse proxies, unaccounted microservices, missing
   CDN layer) are themselves Low findings.

5. **Findings** — grouped by the 12 CSWG review areas, sorted by severity
   within each group. Every review area must appear — either with findings,
   an explicit "no issues found" statement, or (PR mode only) "not touched
   by this PR" where the changed files have no relevance to the area.

6. **CSWG Readiness Assessment** — pass/fail table covering each of the
   12 areas:

```
| # | Area | Status | Key Finding | Remediation Required |
```

Status values: PASS / FAIL / PARTIAL — with one-line justification.

7. **Remediation Plan** — prioritised by CSWG impact:
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

- [ ] Audit Target section populated (Technology Stack, Deployment Context, Audit Focus, Out of Scope, Calibration note)
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
- [ ] Report opened and header block verified to contain no unreplaced `<...>` placeholder tokens
- [ ] No source files were edited, created, or deleted
- [ ] No remediation steps were applied to the codebase

---

## Appendix: NCSC CAF v4.0 Objective Mapping

The NCSC Cyber Assessment Framework v4.0 (August 2025) is referenced in this skill's
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
| 1.7     | 2026-05-28 | Deep self-analysis against external standards, developer-calibrated for Node.js/Rails/Android teams. Added Audience Calibration section (developer-oriented, 10–20 findings, sprint-achievable severity). Added Audit Target section to Output Structure (tech stack, deployment context, audit focus, out-of-scope, calibration note). Fixed LLM Top 10 v2.0 numbering (LLM02/LLM06/LLM08 were using v1 numbers); expanded LLM block to cover LLM03 (supply chain), LLM04 (data poisoning), LLM07 (system prompt leakage), LLM08 (vector/embedding weaknesses). Updated NCSC CAF to v4.0 (August 2025) in header and appendix. Updated ISO 27001 to specify 2022 version. Added OAuth 2.1 mandatory requirements to Area 1 (PKCE for all clients, exact URI match, implicit/password grant removal). Added WebAuthn/passkey conditional block to Area 1. Added SAST blind-spot caveat to Area 5 (tools miss ~50% of vulns; clean SAST ≠ secure code). Added platform-specific Rails block to Area 3 (mass assignment, html_safe/raw, ActiveRecord string interpolation, send/constantize RCE, eval, Brakeman). Added Node.js/Express block to Area 3 (exec injection, eval/new Function, require() path traversal, template injection, helmet middleware, unhandled promise rejections). Added CWE-862 (Missing Authorization, CWE Top 25 2025 #5) to Area 11. Added API6:2023 (unrestricted business flow access) to Area 11. Added Mobile Top 10 2024 M2 (supply chain) and M6 (privacy) to Android block in Area 9. Added deployed-state check to Threat Model section. Fixed Output Structure section numbering (1→7). Added Audit Target checklist item to Completeness Check. |
| 1.6     | 2026-05-28 | Applied Copilot review findings: fixed malformed invocation table row (moved trailing note out of table); added filename sanitisation rule for project/branch components; added optional `Scope` field to finding format for PR extended-scope labelling; added re-audit modifier detection heuristic; added ASVS v5.0.0 alignment note to Frameworks Referenced. Coverage gaps: Area 3 — HTTP request smuggling (CWE-444), DOM-based XSS (CWE-79), prototype pollution (CWE-1321 for JS/Node.js), ReDoS (CWE-1333); Area 2 — CWE-312/CWE-319, data masking/tokenisation; Area 6 — TLS version enforcement, CWE-295, WebSocket origin validation, mTLS; Area 7 — log injection (CWE-117); Area 9 — web cache poisoning; Area 10 — timing attacks (CWE-208), KDF recommendations, nonce/IV reuse (CWE-330/GCM), padding oracle; Area 11 — GraphQL security (conditional); Area 12 — GITHUB_TOKEN over-permission, pull_request_target fork PR exposure, environment vs. repo secrets. Enhancements: chained findings guidance added to Severity Definitions; Mode B large-PR prioritisation rule (>20 files); Done When placeholder validation step; updated description frontmatter. |
| 1.5     | 2026-05-26 | Updated to OWASP Top 10 2025 (replaced all 2021 references); fixed SSRF OWASP citation (A10:2021 retired — no standalone 2025 category, note added); added CSRF (CWE-352, A01:2025) to Area 1; added A10:2025 Mishandling of Exceptional Conditions with CWE-636 Failing Open and CWE-209 to Area 8; added Path Traversal (CWE-22), Insecure Deserialization (CWE-502, A08:2025), Open Redirect (CWE-601), and conditional memory safety block (CWE-787/CWE-125/CWE-416/CWE-190 for memory-unsafe languages) to Area 3; extended JWT token lifecycle bullet (alg:none, RS256/HS256 key-confusion, jku/kid injection) in Area 1; specified CVSS:3.1 vector format with v4.0 note in Finding Format; updated CWE Top 25 → CWE Top 25 2025 in Frameworks Referenced; added CWE-639 to IDOR bullet in Area 11; enumerated HTTP security headers explicitly in Area 9; added supply chain elevation note (A03:2025 is #3) to Area 5; updated OWASP LLM Top 10 to v2.0 (2025) in Area 12 and Frameworks Referenced. |
| 1.4     | 2026-05-26 | Removed disable-model-invocation frontmatter flag (prose guard is sufficient; flag had ambiguous harness semantics). Made CVSS base score required for Critical/High findings, explicitly omitted for Medium/Low. Added re-audit regression modifier (-1 if zero prior findings remediated). Added conditional OWASP MASVS v2.0 / Mobile Top 10 2024 coverage to Area 9 for Android/iOS projects. Added OWASP MASVS v2.0 and Mobile Top 10 2024 (if mobile) to Frameworks Referenced header template. Updated Completeness Check with CVSS and mobile coverage checklist items. Updated grading rubric note to reference all three modifiers. |
| 1.3     | 2026-05-26 | PR mode argument syntax updated: bare PR number (`123`) and GitHub PR URL (`https://github.com/.../pull/123`) now accepted directly — the `pr` prefix is no longer required. |
| 1.2     | 2026-05-26 | Added two invocation modes: Mode A (full local repo audit, default) and Mode B (PR audit via `pr <number>`). PR mode fetches PR metadata and diff via `gh pr`, reads changed files in full, scopes findings to the PR with clearly labelled extended-scope context findings, and adds a Scope Statement below the header. Header block now includes Audit Mode and Audit Scope fields. Completeness check updated for PR mode. |
| 1.1     | 2026-05-26 | Updated OWASP ASVS reference to v5.0.0 (released May 2025, major version). Added OWASP API Security Top 10 2023 to frameworks and coverage across Areas 3, 5, 11, 12 (API3/BOPLA, API4/resource exhaustion, API7/SSRF, API9/shadow APIs, API10/unsafe API consumption). Added SSRF, XXE/CWE-611, TOCTOU/CWE-362 to Area 3. Expanded OAuth 2.0/OIDC attack vectors in Area 1. Added IaC and container security to Area 9. Added supply chain security (SLSA, SBOM, CI/CD pinning, dependency confusion) to Area 12. Named specific secret scanning tools (trufflehog, gitleaks) in Area 4. Named SAST and container/SCA tools (semgrep, trivy, snyk, etc.) in Area 5. Added CVSS base score and Status fields to finding format. Added Low-volume grading modifier (-1 if 10+ Lows). Added Retest Guidance to Remediation Plan section. Added STRIDE per-element methodology to Threat Model section. Added conditional LLM/AI coverage (OWASP LLM Top 10 2025) to Area 12. Fixed git remote contradiction in header template. Added Report Version field and full Low count to Management Summary. Added NCSC CAF objective mapping appendix. Added OWASP LLM Top 10 2025 to Frameworks Referenced. |
| 1.0     | (original) | Initial release — 12 CSWG review areas, severity definitions, grading rubric, finding format, output structure, completeness checklist. |

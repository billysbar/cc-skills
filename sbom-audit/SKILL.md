---
name: sbom-audit
description: Run an SBOM and supply-chain security audit on an npm/Node.js project
disable-model-invocation: true
user-invocable: true
argument-hint: "[PR-number-or-branch]"
---

# SBOM & Supply Chain Security Audit

> **IMPORTANT — REPORT ONLY.**
> This skill produces an audit report. It does **not** make any code changes.
> Do not edit, create, or delete any source files. All findings and recommendations
> are written to the output report exclusively.

**Scope argument**: $ARGUMENTS
- If a PR number (e.g. `228`) or branch name is provided, Phase 4 version-bump checks
  are narrowed to that diff. Phases 1–3 always run against the full repo.
- If empty, audit the full repo on the current branch.

**Output directory**: current working directory
**Output filename**: `YYYY-MM-DD-<project-name>-<branch>-SBOM-AUDIT.md`

Derive repository metadata (project name, branch, commit SHA, git user) from
conversation context and local git. Do NOT run `git remote` commands.
Use `git rev-parse HEAD` and `git branch --show-current` if not already in context.

---

## Ground Rules

- Run all checks in read-only mode. No installs, no mutations, no `npm install`.
- State what IS true, not what "could" or "might" happen.
  - NO: "This package may contain malware"
  - YES: "Package `xyz@1.0.1` contains a `postinstall` script that executes `curl | bash`"
- Critical/High findings require a concrete exploit scenario: attacker action → precondition → observable impact.
- Identify root causes, not symptoms.
- If a check cannot be run (missing binary, missing file), record it as a gap finding at Low severity and continue.

---

## Phase 0 — Context Gathering

Collect and record:
- Project name (from `package.json` `.name`)
- Current branch and full commit SHA
- Node.js version: `node --version`
- npm version: `npm --version`
- Whether `node_modules/` is present (affects which checks are possible)
- Scope: Full repo or PR/branch diff (from $ARGUMENTS)

If $ARGUMENTS is a number, treat it as a GitHub PR number and note it in the header.
If $ARGUMENTS is a string, treat it as a branch name.

---

## Phase 1 — Lockfile & Dependency Pinning

Run each check and record the result (PASS / FAIL / WARN / SKIP):

### 1.1 Lockfile committed
```bash
git status --short package-lock.json
```
- PASS: file is tracked and unmodified
- FAIL: file is missing from git, untracked, or in `.gitignore`
- WARN: file has uncommitted modifications

### 1.2 Lockfile version (integrity)
```bash
jq '.lockfileVersion' package-lock.json
```
- v3 (npm 7+): uses SHA-512 integrity hashes — PASS
- v2: mixed format — WARN
- v1 (npm 6): SHA-1 only — FAIL

### 1.3 npm ci dry-run
```bash
npm ci --dry-run 2>&1 | tail -5
```
- Exit 0: PASS
- Non-zero: FAIL — record error output as finding

### 1.4 Non-registry resolved URLs
```bash
jq -r '[paths(type == "object" and has("resolved")) as $p | getpath($p) | .resolved | select(. != null and (startswith("https://registry.npmjs.org") | not) and startswith("http"))] | unique[]' package-lock.json 2>/dev/null | head -30
```
- Any result: FAIL per URL — record package path and URL
- No results: PASS

### 1.5 Lockfile excluded from .gitignore
```bash
grep -E '^/?package-lock\.json' .gitignore 2>/dev/null
```
- Match found: CRITICAL — lockfile is gitignored, integrity checking is impossible
- No match: PASS

---

## Phase 2 — SBOM Generation & CISA Compliance

### 2.1 Generate CycloneDX SBOM
```bash
npm sbom --format=cyclonedx --json 2>&1
```
If `npm sbom` is unavailable (exit non-zero or "Unknown command"):
- Record as gap finding: "npm sbom requires npm ≥ 9; current version is X"
- Recommend: `npx @cyclonedx/cyclonedx-npm --output-format JSON`
- SKIP remaining Phase 2 checks

### 2.2 Generate SPDX SBOM
```bash
npm sbom --format=spdx --json 2>&1
```

### 2.3 CISA 2025 Minimum Elements check
Validate both SBOM outputs contain:

| Element | CycloneDX field | SPDX field | Required |
|---------|----------------|------------|----------|
| Component name | `components[].name` | `packages[].name` | Yes |
| Component version | `components[].version` | `packages[].versionInfo` | Yes |
| Unique identifier | `components[].bom-ref` | `packages[].SPDXID` | Yes |
| Dependency relationships | `dependencies[]` | `relationships[]` | Yes |
| Hash / integrity | `components[].hashes` | `packages[].checksums` | Yes |
| Supplier | `components[].supplier` | `packages[].supplier` | Yes |
| License | `components[].licenses` | `packages[].licenseConcluded` | Yes |
| Author/tool | `metadata.tools` | `creationInfo.creators` | Yes |

Record: total component count, unique license types found, any missing minimum elements.

---

## Phase 3 — Known Vulnerability Scan

### 3.1 npm audit
```bash
npm audit --json 2>/dev/null
```
Parse output:
- Extract `vulnerabilities` object — count by severity: critical, high, moderate, low, info
- For each critical/high: extract `name`, `severity`, `via[].source`, `fixAvailable`, `range`
- If `npm audit` exits non-zero due to vulnerabilities (not an error), still parse the JSON

### 3.2 OSV-Scanner
```bash
which osv-scanner && osv-scanner --format json . 2>/dev/null || echo "osv-scanner not installed"
```
If installed:
- Parse results; cross-reference against npm audit findings by package name
- Flag any OSV findings not present in npm audit output (tooling gap)

If not installed:
- Record as Low gap finding; recommend: `brew install osv-scanner` or GitHub release binary

### 3.3 Deduplication
Merge findings from 3.1 and 3.2. For each package, report the highest severity seen
across both tools. Note the source tool(s) for each finding.

---

## Phase 4 — Supply Chain Attack Vector Detection

### 4.1 Lifecycle script inventory
```bash
# In node_modules (if present)
find node_modules -maxdepth 2 -name "package.json" -not -path "*/node_modules/*/node_modules/*" \
  -exec jq -r 'select(.scripts | (has("preinstall") or has("postinstall") or has("prepare") or has("postprepare"))) | "\(.name)@\(.version): \(.scripts | to_entries[] | select(.key | test("install|prepare")) | "\(.key)=\(.value)")"' {} \; 2>/dev/null | sort | head -50
```
For each package with lifecycle scripts:
- Record package name, version, script name, and command
- Flag as High if the command contains: `curl`, `wget`, `fetch`, `http`, `exec`, `eval`, `spawn`, `child_process`, `require('fs')`, or base64 decode patterns
- Flag as Moderate for any other postinstall/preinstall (attack surface exists even if benign)

### 4.2 Native binary / gyp packages
```bash
find node_modules -maxdepth 3 -name "binding.gyp" 2>/dev/null | sed 's|/binding.gyp||' | sed 's|node_modules/||' | sort -u | head -20
```
Record all native-addon packages. These download or compile binaries at install time
and have elevated attack surface. Flag as Low unless also flagged in 4.1.

### 4.3 Major-version bump detection (PR/diff scope)

If $ARGUMENTS supplied — compare lockfile on current branch vs base branch:
```bash
# Get base branch (default: develop or main)
BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||' || echo "develop")
git show origin/${BASE}:package-lock.json > /tmp/sbom-audit-base-lock.json 2>/dev/null || echo "base lockfile unavailable"
```
If base lockfile available:
```bash
# Extract name@version pairs from both lockfiles and diff
jq -r '.packages | to_entries[] | select(.key != "") | "\(.key | split("/")[-1])@\(.value.version)"' package-lock.json | sort > /tmp/sbom-audit-current.txt
jq -r '.packages | to_entries[] | select(.key != "") | "\(.key | split("/")[-1])@\(.value.version)"' /tmp/sbom-audit-base-lock.json | sort > /tmp/sbom-audit-base.txt
diff /tmp/sbom-audit-base.txt /tmp/sbom-audit-current.txt
```
For each changed package:
- Parse old and new version using semver
- Flag Major bump (X.y.z → X+1.y.z) as High
- Flag Minor bump as Moderate
- Flag new additions (not in base) as Moderate
- Flag removals as Low

**PR-specific named packages** (always check explicitly even without $ARGUMENTS):

`sharp`:
```bash
# Current version
jq -r '.packages["node_modules/sharp"].version // .dependencies.sharp.version' package-lock.json
# Check if pre-built binary source changed (libvips version)
jq -r '.packages["node_modules/sharp"] | {"version":.version,"engines":.engines}' package-lock.json 2>/dev/null
```

`grunt-sass`:
```bash
jq -r '.packages["node_modules/grunt-sass"] | {"version":.version,"peerDependencies":.peerDependencies}' package-lock.json 2>/dev/null
# Check whether sass (Dart Sass) or node-sass is resolved
jq -r 'if .packages["node_modules/node-sass"] then "node-sass: \(.packages["node_modules/node-sass"].version)" else "node-sass: not present" end' package-lock.json 2>/dev/null
jq -r 'if .packages["node_modules/sass"] then "sass (Dart): \(.packages["node_modules/sass"].version)" else "sass (Dart): not present" end' package-lock.json 2>/dev/null
```

### 4.4 Scoped package / dependency confusion check
```bash
# List all @org/ scoped direct dependencies
jq -r '.dependencies | keys[] | select(startswith("@"))' package.json 2>/dev/null
```
For each scoped package, note that dependency confusion attacks substitute a
public package with the same name. Flag as Low with recommendation to verify
`.npmrc` registry scope configuration.

---

## Phase 5 — CI/CD & SDLC Posture

### 5.1 CI configuration check
```bash
find . -maxdepth 3 \( -name ".travis.yml" -o -name "Jenkinsfile" -o -name "*.yml" -path "*/.github/workflows/*" \) 2>/dev/null
```
For each CI config found:
- Check for `npm ci` (PASS) vs `npm install` (WARN — doesn't enforce lockfile)
- Check for `npm audit` or SCA tool invocation (PASS if present; Low gap if absent)
- Check for multi-OS/arch matrix (flag if Linux-only for a project with native binaries)

### 5.2 .gitignore audit for lockfile
Already covered in Phase 1.5. Reference finding here.

### 5.3 Dependabot / Renovate configuration
```bash
cat .github/dependabot.yml 2>/dev/null || echo "not found"
cat renovate.json 2>/dev/null || echo "not found"
```
- Present and enables npm ecosystem: PASS
- Absent: Low gap finding

---

## Report Assembly

Write the complete report to `YYYY-MM-DD-<project-name>-<branch>-SBOM-AUDIT.md`
in the current working directory.

---

### Section 1: Header

```
Repository Name       : <package.json name>
Branch / Commit       : <branch> (<full SHA>)
Scope                 : Full repo | PR #<n> | Branch <name>
Node Version          : <x.y.z>
npm Version           : <x.y.z>
Analysis Date         : <YYYY-MM-DD>
Reviewer              : <git config user.name>
Overall Risk Rating   : <Critical | High | Moderate | Low>
CISA SBOM Compliance  : <Compliant | Partial | Non-compliant>
Frameworks Referenced : CISA 2025 SBOM Minimum Elements, OWASP CI/CD Top 10, NIST SSDF, OSV.dev schema
```

**Overall Risk Rating** derived from highest severity finding:
- Any Critical → Critical
- Any High, no Critical → High
- Any Moderate, no High/Critical → Moderate
- Low only → Low

**CISA SBOM Compliance**:
- All 8 minimum elements present in generated SBOM → Compliant
- 1–3 elements missing → Partial
- npm sbom unavailable or 4+ elements missing → Non-compliant

---

### Section 2: Management Summary

- Finding count table: Critical | High | Moderate | Low (columns), Phase 1–5 (rows)
- 2–3 sentences on overall supply-chain posture in plain language (no code references)
- Top 5 risks ranked by business impact — one sentence each, no line numbers
- Overall Risk Rating with one-line justification

---

### Section 3: SBOM Summary

| Metric | Value |
|--------|-------|
| Total components | N |
| Direct dependencies | N |
| Transitive dependencies | N |
| Unique licenses | list |
| SBOM format generated | CycloneDX vX / SPDX vX |
| CISA minimum elements | N/8 present |
| Missing elements | list or "none" |

---

### Section 4: Findings

Group by phase. Within each group, sort Critical → High → Moderate → Low.
Every phase must appear — either with findings or an explicit "no issues found" statement.

**Finding format:**
```
### [Short Title]
**Severity**: Critical / High / Moderate / Low
**Attack Pattern**: [e.g. Lifecycle Script Abuse | Lockfile Tampering | Known CVE | Dependency Confusion | Major Version Bump]
**OWASP**: CICD-SEC-X (name) — or N/A
**Package / Location**: <name@version> or <filename>
**What happens**: [Observed fact]
**Exploit scenario**: [Required for Critical/High: attacker action → precondition → observable impact]
**Recommendation**: [Specific, actionable fix]
```

**Severity definitions:**
- **Critical**: Active exploit signal; package known-compromised; lockfile completely absent
- **High**: CVE CVSS ≥ 7.0; lifecycle script with outbound network call; lockfile integrity failure
- **Moderate**: CVE CVSS 4.0–6.9; non-registry resolved URL; major version bump without review; new lifecycle script
- **Low**: Best-practice gap; missing SCA in CI; no Dependabot; native binary (no other flags)

---

### Section 5: Jira Checklist (PZBPAPI-3799)

Answer each item explicitly — PASS, FAIL, or N/A with one-line evidence:

| # | Question | Result | Evidence |
|---|----------|--------|----------|
| 1 | Is `package-lock.json` pinned and committed? | | |
| 2 | Did `sharp` major-version change? | | old → new version |
| 3 | Did `grunt-sass` move from `node-sass` to `sass`? | | present/absent in lockfile |
| 4 | Any new `postinstall`, `prepare`, or lifecycle scripts? | | list or "none" |
| 5 | Any new packages with maintainers changed recently? | | gap if registry API unavailable |
| 6 | Was a software composition analysis run? | | npm audit / OSV-Scanner result |
| 7 | Were builds tested on all CI targets? | | CI matrix finding |

For item 5 (maintainer change): if npm registry API is accessible, check the
5 most recently-added direct dependencies. Otherwise note this as a manual check gap.

---

### Section 6: Remediation Plan

Prioritised by risk:

- **P0 — Fix before any merge** (Critical/High): specific JIRA-ready action titles
- **P1 — Fix this sprint** (Moderate with high visibility)
- **P2 — Improve when possible** (remaining Moderate/Low)

Use imperative, concise titles: "Pin package-lock.json in .gitignore exclusion",
"Replace npm install with npm ci in .travis.yml", etc.

---

## Completeness Check

Before finalising, verify:
- [ ] Header fully populated (no placeholder values)
- [ ] All 5 phases addressed — findings or explicit "no issues found"
- [ ] All Critical/High findings include exploit scenario
- [ ] Overall Risk Rating matches highest severity finding
- [ ] CISA compliance verdict matches Section 3 element count
- [ ] Jira Checklist (Section 5) has all 7 rows answered
- [ ] Remediation Plan covers all Critical/High findings
- [ ] No source files were edited, created, or deleted

## Done When

- [ ] `.md` file written to CWD with correct filename
- [ ] No source files modified
- [ ] No packages installed or removed

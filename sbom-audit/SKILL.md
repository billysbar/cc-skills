---
name: sbom-audit
description: Run an SBOM and supply-chain security audit on any codebase (polyglot — npm, Ruby, Python, Go, Rust, Java, .NET, Android, Container)
user-invocable: true
argument-hint: "[PR-number-or-branch]"
---

# SBOM & Supply Chain Security Audit

> **IMPORTANT — REPORT ONLY.**
> This skill produces an audit report. It does **not** make any code changes.
> Do not edit, create, or delete any source files. All findings and recommendations
> are written to the output report exclusively.

**Scope argument**: $ARGUMENTS
- If a PR number (e.g. `228`) or branch name is provided, Phase 4 version-bump and
  typosquatting checks are narrowed to that diff. Phases 1–3 always run against the full repo.
- If empty, audit the full repo on the current branch.

**Output directory**: current working directory
**Output filename**: `YYYY-MM-DD-<project-name>-<branch>-SBOM-AUDIT.md`

Derive repository metadata (project name, branch, commit SHA, git user) from
conversation context and local git. Do NOT run `git remote` commands.
Use `git rev-parse HEAD` and `git branch --show-current` if not already in context.

---

## Ground Rules

- Run all checks in read-only mode. No installs, no mutations.
- State what IS true, not what "could" or "might" happen.
  - NO: "This package may contain malware"
  - YES: "Package `xyz@1.0.1` contains a `postinstall` script that executes `curl | bash`"
- Critical/High findings require a concrete exploit scenario: attacker action → precondition → observable impact.
- Identify root causes, not symptoms.
- If a check cannot be run (missing binary, missing file), record it as a gap finding at Low severity and continue.
- In multi-ecosystem projects, label every ecosystem-specific finding with `[Ecosystem: <name>]`.

---

## Phase 0 — Ecosystem Detection & Tool Inventory

Phase 0 is the intelligence layer. Its output drives every subsequent phase.
Run all detection steps before proceeding to Phase 1.

### 0.1 Detect Ecosystem Manifests

Check for the following files and record which ecosystems are present:

```bash
# Node.js / npm
[ -f package.json ] && echo "npm: package.json found"
[ -f package-lock.json ] && echo "npm: package-lock.json (npm)"
[ -f yarn.lock ] && [ ! -f package-lock.json ] && echo "npm: yarn.lock (yarn)"
[ -f pnpm-lock.yaml ] && echo "npm: pnpm-lock.yaml (pnpm)"

# Ruby
[ -f Gemfile ] && echo "ruby: Gemfile found"

# Python
[ -f pyproject.toml ] && echo "python: pyproject.toml found"
[ -f requirements.txt ] && echo "python: requirements.txt found"
[ -f Pipfile ] && echo "python: Pipfile found"

# Go
[ -f go.mod ] && echo "go: go.mod found"

# Rust
[ -f Cargo.toml ] && echo "rust: Cargo.toml found"

# Java / Maven
[ -f pom.xml ] && echo "java-maven: pom.xml found"

# Java / Gradle (includes Android)
[ -f build.gradle ] || [ -f build.gradle.kts ] && echo "java-gradle: build.gradle found"
find . -name "settings.gradle" -o -name "settings.gradle.kts" -maxdepth 2 2>/dev/null

# .NET / C#
find . -name "*.csproj" -o -name "*.sln" -maxdepth 3 2>/dev/null | head -3

# Container
find . -name "Dockerfile*" -not -path "*/node_modules/*" -maxdepth 4 2>/dev/null | head -5
[ -f docker-compose.yml ] || [ -f docker-compose.yaml ] && echo "container: docker-compose found"
```

Build `ECOSYSTEMS_FOUND` list. If zero ecosystems detected, note it — still run Phase 5.

### 0.2 Probe Available SBOM Generation Tools

```bash
which cdxgen 2>/dev/null && cdxgen --version 2>/dev/null || echo "cdxgen: not installed"
which syft 2>/dev/null && syft version 2>/dev/null || echo "syft: not installed"
which trivy 2>/dev/null && trivy --version 2>/dev/null | head -1 || echo "trivy: not installed"
```

Set `SBOM_GEN_TOOL` to the first available: cdxgen → syft → trivy → ecosystem-native → none.

If none are available, record as Critical gap finding: "No universal SBOM generation tool found. Install cdxgen (`npm install -g @cyclonedx/cdxgen`), syft (`brew install syft`), or trivy (`brew install trivy`)."

### 0.3 Probe Available CVE Scan Tools

```bash
which osv-scanner 2>/dev/null && osv-scanner --version 2>/dev/null || echo "osv-scanner: not installed"
which grype 2>/dev/null && grype version 2>/dev/null | head -1 || echo "grype: not installed"
which trivy 2>/dev/null || echo "trivy: not installed"
# Ecosystem-native (check only for detected ecosystems)
which npm 2>/dev/null && npm --version || true
which bundle 2>/dev/null && bundle --version || true
which pip 2>/dev/null && pip --version || true
which govulncheck 2>/dev/null || true
which cargo 2>/dev/null && cargo --version || true
```

### 0.4 Project Metadata

```bash
git rev-parse HEAD 2>/dev/null
git branch --show-current 2>/dev/null
git config user.name 2>/dev/null
```

Determine project name from: `package.json` `.name`, `Gemfile` directory name, `pyproject.toml` `[project].name`, `go.mod` module path, `pom.xml` `<artifactId>`, or the repository directory name as fallback.

Collect runtime versions only for detected ecosystems:
```bash
# npm
[ -f package.json ] && node --version 2>/dev/null; npm --version 2>/dev/null
# Ruby
[ -f Gemfile ] && ruby --version 2>/dev/null; bundle --version 2>/dev/null
# Python
[ -f pyproject.toml ] || [ -f requirements.txt ] && python3 --version 2>/dev/null
# Go
[ -f go.mod ] && go version 2>/dev/null
# Rust
[ -f Cargo.toml ] && rustc --version 2>/dev/null
```

### 0.5 Emit Audit Configuration

Record this block verbatim — it becomes **Section 0** of the report:

```
Ecosystems detected   : <list from 0.1, or "None">
Package manager(s)    : <e.g. npm v10.x, bundler v2.x>
node_modules present  : Yes / No  (npm only)
Lockfiles present     : <list>
SBOM generation tool  : <tool name + version, or "None available — see gap finding">
CVE scan tools        : <list of available tools>
Scope                 : Full repo | PR #<n> | Branch <name>
Branch / Commit       : <branch> (<SHA>)
Analysis Date         : <YYYY-MM-DD>
```

---

## Phase 1 — Lockfile & Dependency Pinning

### Universal checks (all ecosystems)

**1.U1 — Lockfile committed**
For each lockfile found in Phase 0:
```bash
git status --short <lockfile>
```
- PASS: tracked and unmodified
- FAIL: untracked, missing, or in `.gitignore`
- WARN: uncommitted modifications

**1.U2 — Lockfile not gitignored**
```bash
grep -rE '(package-lock\.json|yarn\.lock|pnpm-lock\.yaml|Gemfile\.lock|poetry\.lock|Pipfile\.lock|go\.sum|Cargo\.lock|gradle\.lockfile|packages\.lock\.json)' .gitignore 2>/dev/null
```
Any match: **CRITICAL** — lockfile is gitignored, integrity checking is impossible.

### Ecosystem-specific checks (run only for detected ecosystems)

**npm (package-lock.json)**
```bash
# Lockfile version / hash level
jq '.lockfileVersion' package-lock.json
# v3 (npm ≥ 7): SHA-512 → PASS; v2: mixed → WARN; v1: SHA-1 only → FAIL

# Consistency check (requires clean environment — no prior node_modules)
npm ci --dry-run 2>&1 | tail -5
# Exit 0: PASS. Non-zero: FAIL — record error output.

# Non-registry resolved URLs
jq -r '[paths(type == "object" and has("resolved")) as $p | getpath($p) | .resolved
  | select(. != null and (startswith("https://registry.npmjs.org") | not) and startswith("http"))]
  | unique[]' package-lock.json 2>/dev/null | head -20
```

**Ruby (Gemfile.lock)**
```bash
bundle check 2>&1
# Checks that Gemfile.lock satisfies the Gemfile. Exit 0: PASS.

# Non-rubygems.org sources
grep -A2 "^GEM" Gemfile.lock | grep "remote:" | grep -v "rubygems.org"
```

**Python/poetry (poetry.lock)**
```bash
poetry check --lock 2>&1
# Exit 0: PASS. Verifies lockfile is consistent with pyproject.toml.
```

**Python/pip (requirements.txt only — no lockfile)**
Flag as Moderate gap: `requirements.txt` without a lockfile (`poetry.lock` or `pip-compile`-generated `requirements.txt`) does not pin transitive dependencies or content hashes. Recommend `pip-compile --generate-hashes` or migrating to Poetry.

**Go (go.sum)**
```bash
go mod verify 2>&1
# Verifies downloaded modules match go.sum hashes. Exit 0: PASS.
```

**Rust (Cargo.lock)**
```bash
git status --short Cargo.lock
# Cargo.lock committed: PASS. Check workspace members too.
```

**Gradle (gradle.lockfile)**
```bash
find . -name "*.lockfile" -path "*/gradle/*" 2>/dev/null | head -5
# Present: PASS. Absent: WARN — Gradle dependency locking disabled.
```

**.NET (packages.lock.json)**
```bash
find . -name "packages.lock.json" -maxdepth 4 2>/dev/null | head -5
# Check RestorePackagesWithLockFile in .csproj or Directory.Build.props
grep -r "RestorePackagesWithLockFile" . --include="*.props" --include="*.csproj" 2>/dev/null
```

---

## Phase 2 — SBOM Generation & Standards Compliance

### 2.1 Universal SBOM Generation (CycloneDX)

Use `SBOM_GEN_TOOL` set in Phase 0. Try in order:

```bash
# cdxgen (preferred — auto-detects ecosystem)
cdxgen -o sbom-audit-bom.json --spec-version 1.6 2>&1 | tail -10

# syft (fallback)
syft . -o cyclonedx-json=sbom-audit-bom.json 2>&1 | tail -5

# trivy (fallback)
trivy fs --format cyclonedx --output sbom-audit-bom.json . 2>&1 | tail -5
```

Ecosystem-native (only if no universal tool available):
- npm: `npm sbom --format=cyclonedx --json 2>&1` — requires npm ≥ 9
- Ruby: `cyclonedx-ruby --output sbom-audit-bom.xml 2>&1`
- Python: `cyclonedx-py environment --output-format JSON 2>&1`
- Go: `cyclonedx-gomod mod -json -output sbom-audit-bom.json 2>&1`

If no tool produces a SBOM: record Critical gap finding and skip all checks in this phase that depend on SBOM output.

### 2.2 SPDX Output

```bash
# cdxgen
cdxgen -o sbom-audit-bom.spdx.json --output-format spdx --spec-version 2.3 2>&1 | tail -5

# syft
syft . -o spdx-json=sbom-audit-bom.spdx.json 2>&1 | tail -5

# trivy
trivy fs --format spdx-json --output sbom-audit-bom.spdx.json . 2>&1 | tail -5
```

### 2.3 CISA 2025 Minimum Elements Validation

Validate the generated SBOM (CycloneDX and/or SPDX) against these 8 required elements:

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

```bash
# Quick completeness check (CycloneDX JSON)
jq '{
  components: (.components | length),
  with_version: [.components[] | select(.version)] | length,
  with_hashes: [.components[] | select(.hashes | length > 0)] | length,
  with_supplier: [.components[] | select(.supplier)] | length,
  with_license: [.components[] | select(.licenses | length > 0)] | length,
  has_dependencies: (.dependencies | length > 0),
  has_tools: (.metadata.tools | length > 0)
}' sbom-audit-bom.json 2>/dev/null
```

Compliance:
- All 8 elements present → Compliant
- 1–3 elements missing → Partial
- No SBOM generated or 4+ missing → Non-compliant

### 2.4 Provenance & Signing Checks

**npm (≥ 9.5)**
```bash
npm audit signatures 2>/dev/null || echo "npm audit signatures requires npm ≥ 9.5"
```
Any direct dependency lacking a provenance attestation on a Critical/High severity chain: flag as Moderate.

**All ecosystems** — check for sigstore/cosign attestation files:
```bash
find . -name "*.intoto.jsonl" -o -name "*.sigstore" -o -name "*.att" -maxdepth 4 2>/dev/null | head -10
which cosign 2>/dev/null && echo "cosign available" || echo "cosign not installed"
```
If cosign is absent across a non-trivial codebase: flag as Low gap.

Record provenance coverage metric: how many direct dependencies have signed provenance attestations vs. total direct dependencies.

### 2.5 VEX Document Presence

```bash
find . \( -name "*.vex" -o -name "vex.json" -o -name ".openvex.json" \
  -o -name "*.csaf" -o -name "csaf.json" \) -maxdepth 4 2>/dev/null | head -10
```

- VEX document(s) found: parse for CVE IDs with statements; note count; pass statements to Phase 3.6
- No VEX found: record as Low gap finding with recommendation to adopt OpenVEX (`openvex.io`) or CycloneDX VEX notation to suppress false-positive CVEs

---

## Phase 3 — Known Vulnerability Scan

Run all available scanners. Merge results in step 3.5.

### 3.1 osv-scanner (universal, lockfile-based)
```bash
osv-scanner --format json . 2>&1
```
Parses all lockfiles present. Records results by ecosystem. If not installed: Low gap finding; `brew install osv-scanner`.

### 3.2 grype (SBOM-based, if SBOM generated)

Run Grype twice to produce two distinct result sets:

```bash
# Run 1: CI-gate scan — only CVEs with an available fix (mirrors ci.yml only-fixed: true)
grype sbom:sbom-audit-bom.json --output json --only-fixed 2>&1 > /tmp/grype-ci.json

# Run 2: full scan — all CVEs including those with no fix available
grype sbom:sbom-audit-bom.json --output json 2>&1 > /tmp/grype-full.json
```

Cross-reference with NVD, GHSA, and OSV. Report both result sets separately in Section 4 and
Section 5: CI-gate findings (what would block a PR in production) vs. additional findings
(informational — no fix available, CI would not block on these). If not installed: Low gap
finding; `brew install grype`.

### 3.3 trivy (if available and not already used as SBOM generator)
```bash
trivy fs --scanners vuln --format json . 2>&1 | head -200
```

### 3.4 Ecosystem-native CVE tools (run for each detected ecosystem)

**npm**
```bash
npm audit --json 2>&1
```
Parse `vulnerabilities` object — count by severity. For each critical/high: extract `name`, `severity`, `via[].source`, `fixAvailable`, `range`, and CVSS base score from `.cvss.score` or `.via[].cvss.score`.

**Ruby**
```bash
bundler-audit check --update 2>&1
```
If not installed: `gem install bundler-audit`.

**Python**
```bash
pip audit --format json 2>&1
```
If not installed: `pip install pip-audit`.

**Go**
```bash
govulncheck ./... 2>&1
```
If not installed: `go install golang.org/x/vuln/cmd/govulncheck@latest`.

**Rust**
```bash
cargo audit --json 2>&1
```
If not installed: `cargo install cargo-audit`.

**.NET**
```bash
dotnet list package --vulnerable --format json 2>&1
```

### 3.5 Deduplication

Merge all findings by CVE ID (e.g. CVE-2021-23337). For each unique CVE:
- Use the highest severity seen across all tools
- Record which tool(s) flagged it: `[osv-scanner, npm audit]`
- Include CVSS base score where available

### 3.6 VEX-Aware Filtering

If VEX documents were found in Phase 2.5:
- For each CVE in the merged findings, check if a VEX statement exists
- Mark as `[VEX: not-affected]` or `[VEX: mitigated]` — exclude from severity counts
- Still list VEX-suppressed CVEs in the findings table for transparency with a `[VEX suppressed]` tag
- Add VEX Status row to Section 4 (SBOM Summary): `N CVEs suppressed by VEX / M total CVEs found`

### 3.7 Grype Suppression File Audit

```bash
if [ -f .grype.yaml ]; then
  echo ".grype.yaml found — $(grep -c 'vulnerability:' .grype.yaml) suppression entries"
  cat .grype.yaml
else
  echo ".grype.yaml not present — no Grype suppressions configured"
fi
```

If `.grype.yaml` is present, Grype applies its `ignore` rules automatically during any scan in
Phase 3.2. Suppressed findings are silently excluded from scan output and will not appear in
Phase 3.5 deduplication — they are invisible unless explicitly audited here.

For each `ignore` entry found:
- Report as a finding in Section 5 with tag `[Grype-Suppressed]`
- Severity: as reported by the CVE database for that vulnerability ID
- State the suppression rationale if a comment is present above the entry in the file
- Flag entries with no rationale comment as **Low** (undocumented risk acceptance)
- Record the total suppression count in the SBOM Summary table (Section 4)

Suppression entries are version-pinned — a suppression for `form-data@2.3.3` does not apply
to `form-data@2.5.4`. Cross-reference each pinned version against `package-lock.json` to
confirm the suppressed package is still at the pinned version (i.e. the suppression is active
and not silently stale).

---

## Phase 4 — Supply Chain Attack Vector Detection

### 4.1 Lifecycle Script / Hook Inventory

**npm** — scan lockfile directly (works without `node_modules` present):
```bash
jq -r '.packages | to_entries[]
  | select(.value.scripts | (has("preinstall") or has("postinstall") or has("prepare") or has("postprepare")))
  | "\(.key): \(.value.scripts | to_entries[] | select(.key | test("install|prepare")) | "\(.key)=\(.value)")"
' package-lock.json 2>/dev/null | head -50
```
Flag as **High** if script contains: `curl`, `wget`, `fetch`, `http`, `exec`, `eval`, `spawn`, `child_process`, `require('fs')`, base64 decode patterns.
Flag as **Moderate** for any other postinstall/preinstall (attack surface exists even if benign).

**Ruby** — gem C extensions and post-install hooks:
```bash
# Gems with native extensions (compile C code at install time)
grep -A1 "extensions:" Gemfile.lock 2>/dev/null | head -20
# Check for post_install_message with shell patterns
find . -path "*/gems/*/lib" -name "*.rb" -exec grep -l "post_install_message\|system(\|exec(" {} \; 2>/dev/null | head -10
```

**Python** — setup.py hooks:
```bash
# Check for subprocess / os.system in setup.py
grep -n "subprocess\|os\.system\|os\.popen\|exec(" setup.py 2>/dev/null
# Check build-system in pyproject.toml
grep -A5 "\[build-system\]" pyproject.toml 2>/dev/null
```

**Gradle/Android** — custom exec tasks:
```bash
grep -rn "exec {" --include="*.gradle" --include="*.gradle.kts" . 2>/dev/null | head -20
grep -rn "Runtime.getRuntime().exec\|ProcessBuilder" --include="*.gradle.kts" . 2>/dev/null | head -10
```

### 4.2 Native Binary / Native Addon Detection

```bash
# npm: binding.gyp (Node native addons)
find . -name "binding.gyp" -not -path "*/node_modules/*/node_modules/*" 2>/dev/null | sed 's|/binding.gyp||' | sort -u

# Python: C extensions
find . -name "*.pyx" -o -name "*.pxd" -o -name "*.c" -path "*/python*" 2>/dev/null | head -10

# Ruby: extconf.rb (C extensions)
find . -name "extconf.rb" 2>/dev/null | head -10
```

Flag all native-addon/extension packages as **Low** (elevated attack surface) unless they also trigger findings in 4.1.

### 4.3 Version Bump Detection (diff scope — runs when $ARGUMENTS provided)

Determine base branch:
```bash
BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||' || echo "main")
```

For each detected ecosystem, extract old and new lockfile and diff:

**npm** — compare name@version AND integrity hashes:
```bash
git show origin/${BASE}:package-lock.json > /tmp/sbom-audit-base-lock.json 2>/dev/null
# Version diff
jq -r '.packages | to_entries[] | select(.key != "") | "\(.key | split("/")[-1])@\(.value.version)"' package-lock.json | sort > /tmp/sbom-current-ver.txt
jq -r '.packages | to_entries[] | select(.key != "") | "\(.key | split("/")[-1])@\(.value.version)"' /tmp/sbom-audit-base-lock.json | sort > /tmp/sbom-base-ver.txt
diff /tmp/sbom-base-ver.txt /tmp/sbom-current-ver.txt
# Integrity hash diff (detects registry substitution without version bump)
jq -r '.packages | to_entries[] | select(.key != "") | "\(.key):\(.value.integrity // "NO_HASH")"' package-lock.json | sort > /tmp/sbom-current-hash.txt
jq -r '.packages | to_entries[] | select(.key != "") | "\(.key):\(.value.integrity // "NO_HASH")"' /tmp/sbom-audit-base-lock.json | sort > /tmp/sbom-base-hash.txt
diff /tmp/sbom-base-hash.txt /tmp/sbom-current-hash.txt
```

**Ruby** — diff Gemfile.lock:
```bash
git show origin/${BASE}:Gemfile.lock > /tmp/sbom-base-gemfile.lock 2>/dev/null
diff /tmp/sbom-base-gemfile.lock Gemfile.lock | grep "^[<>]" | grep -E "^\+[[:space:]]+[a-z]" | head -30
# Bundler 2.5+ CHECKSUMS section: diff SHA-256 hashes
```

**Go** — diff go.sum (every entry is a hash):
```bash
git show origin/${BASE}:go.sum > /tmp/sbom-base-go.sum 2>/dev/null
diff /tmp/sbom-base-go.sum go.sum
```

For each changed package, classify:
- Major bump (X.y.z → X+1.y.z): **High**
- Minor bump: **Moderate**
- New addition (not in base): **Moderate**
- Hash changed without version bump: **High** (potential registry substitution attack)
- Removal: **Low**

### 4.4 Dependency Confusion Check

For each ecosystem, verify private namespace / registry isolation:

**npm** — check `.npmrc` for scope-to-registry mappings:
```bash
cat .npmrc 2>/dev/null
# All @org/ scoped packages should map to a private registry
jq -r '.dependencies | keys[] | select(startswith("@"))' package.json 2>/dev/null
```

**Ruby** — Gemfile source ordering:
```bash
grep "^source\|^gem.*:source" Gemfile 2>/dev/null
# Private gems should reference a private source before rubygems.org
```

**Python** — pip index configuration:
```bash
grep -E "index-url|extra-index-url|trusted-host" pip.conf .pip/pip.conf pyproject.toml 2>/dev/null
```

**Maven** — repository configuration in settings.xml:
```bash
cat ~/.m2/settings.xml 2>/dev/null | grep -A3 "<repository>" | head -20
```

Flag as **Low** with recommendation to verify registry isolation; upgrade to **Moderate** if private packages are present without confirmed registry pinning.

### 4.5 Typosquatting Risk Check (newly added packages in diff scope)

For each newly added direct dependency identified in 4.3:
```bash
# npm: check package age and version count via registry API
npm view <package-name> time.created --json 2>/dev/null
npm view <package-name> versions --json 2>/dev/null | jq 'length'

# Ruby: check gem age
gem search <gem-name> --remote --details 2>/dev/null | grep "Authors\|Date"

# Python: check PyPI
curl -s "https://pypi.org/pypi/<package>/json" 2>/dev/null | jq '{created: .info.created, version_count: (.releases | keys | length)}'
```

Flag as **Low** (requires manual confirmation) if a newly added package is:
- Less than 30 days old on the registry, AND
- Has fewer than 5 prior published versions

Note the package name and creation date in the finding. Do not flag existing packages already in the base lockfile.

### 4.6 Integrity Hash Regression

Already covered in 4.3 (hash diff). Explicitly flag any package where the lockfile hash changed between base and current without a corresponding version bump as **High** (potential registry substitution / manifest confusion attack).

---

## Phase 5 — CI/CD & SDLC Posture

### 5.1 CI Configuration Check

```bash
find . -maxdepth 4 \( \
  -name ".travis.yml" \
  -o -name "Jenkinsfile" \
  -o -name ".circleci/config.yml" \
  -o -name ".gitlab-ci.yml" \
  -o -path "*/.github/workflows/*.yml" \
\) 2>/dev/null
```

For each CI config found, check — adapting expected commands to detected ecosystem:
- **Install command**: `npm ci` PASS vs `npm install` WARN; `bundle install --frozen` PASS vs `bundle install` WARN; `pip sync` PASS vs `pip install -r` WARN; `go mod download` with verification PASS
- **SCA tool invocation**: `npm audit`, `bundler-audit`, `osv-scanner`, `grype`, `trivy` in any step → PASS; absent → Low gap
- **Multi-OS/arch matrix**: flag if Linux-only for a project with native binaries
- **Security result upload**: check that steps uploading SCA/SBOM scan results use
  `if: always()`. Without this, a failing scan step prevents upload of the results — the
  exact moment they are needed. Flag absent as **Low** gap.
- **SBOM artifact retention**: check that the generated SBOM is uploaded as a CI artifact
  with explicit retention days. A SBOM generated but not archived provides no audit trail.
  Flag absent as **Low** gap.

### 5.2 GitHub Actions Version Pinning

For each workflow file found:
```bash
grep -n "uses:" .github/workflows/*.yml 2>/dev/null | head -40
```

Classify each `uses:` reference:
- `uses: actions/checkout@abc123def456` (SHA pin): **PASS**
- `uses: actions/checkout@v4` (tag pin): **WARN** — tag can be moved; recommend SHA pinning
- `uses: actions/checkout@main` or `@master` (branch pin): **High** — mutable reference

### 5.3 SLSA Posture Check

```bash
# Provenance attestation files
find . \( -name "*.intoto.jsonl" -o -name "provenance.json" -o -name "provenance.jsonl" \) 2>/dev/null

# SLSA GitHub generator usage
grep -r "slsa-framework/slsa-github-generator" .github/workflows/ 2>/dev/null

# Artifact signing
which cosign 2>/dev/null && echo "cosign available" || echo "cosign not installed"
```

Rate current SLSA posture:
- **Level 0**: No scripted build, no provenance → **High** gap if publishing artefacts
- **Level 1**: Build scripted; some provenance exists → **Moderate** gap if missing
- **Level 2**: Signed provenance from hosted CI build → **Low** gap if missing
- **Level 3**: Hardened build environment, non-falsifiable provenance → reference only; note if present

### 5.4 Container / Dockerfile Supply Chain

**Runs only if Dockerfile was detected in Phase 0.**

```bash
# Find all Dockerfiles
find . -name "Dockerfile*" -not -path "*/node_modules/*" -maxdepth 4 2>/dev/null

# Check for floating (non-digest-pinned) base images
grep -n "^FROM" Dockerfile* 2>/dev/null | grep -v "@sha256:"

# Check for unpinned ARG-based base images
grep -n "^ARG.*IMAGE\|^FROM \$" Dockerfile* 2>/dev/null
```

Classification:
- `FROM node:18-alpine@sha256:abc123` (digest-pinned): **PASS**
- `FROM node:18-alpine` (tag-pinned): **WARN** — tag can be silently updated
- `FROM node:latest` or `FROM node`: **High** — fully mutable

If Trivy is available, recommend base image CVE scan:
```bash
trivy image --format cyclonedx <image-name> 2>&1
```

Also check for secrets or credentials in Dockerfile:
```bash
grep -n "ENV.*PASSWORD\|ENV.*SECRET\|ENV.*KEY\|ENV.*TOKEN" Dockerfile* 2>/dev/null
```

### 5.5 Dependabot / Renovate Configuration

```bash
cat .github/dependabot.yml 2>/dev/null || echo "dependabot.yml: not found"
cat renovate.json 2>/dev/null || cat .renovaterc 2>/dev/null || cat .renovaterc.json 2>/dev/null || echo "renovate config: not found"
```

For each file found, check that all detected ecosystems are covered in the config.
Flag as **Low** gap if no automated dependency update tool is configured.
Flag as **Moderate** if tool is configured but misses one or more detected ecosystems.
For GitHub repositories, also check that the `github-actions` ecosystem block is present —
required to keep SHA-pinned CI actions updated automatically. Without it, action SHA pins
drift to stale versions and require manual updates. Flag absent as **Low** gap.

---

## Report Assembly

Write the complete report to `YYYY-MM-DD-<project-name>-<branch>-SBOM-AUDIT.md`
in the current working directory.

---

### Section 0: Detected Configuration

Paste the Audit Configuration block verbatim from Phase 0.5.

---

### Section 1: Header

```
Repository Name       : <project name>
Branch / Commit       : <branch> (<full SHA>)
Scope                 : Full repo | PR #<n> | Branch <name>
Ecosystems            : <list>
Runtime(s)            : <e.g. Node.js v20.x / Ruby 3.3 / Python 3.12>
SBOM Generation Tool  : <tool + version, or "None — see Critical gap finding">
Analysis Date         : <YYYY-MM-DD>
Reviewer              : <git config user.name>
Overall Risk Rating   : <Critical | High | Moderate | Low>
CISA SBOM Compliance  : <Compliant | Partial | Non-compliant>
Frameworks Referenced : CISA 2025 SBOM Minimum Elements, OWASP CI/CD Top 10 (CICD-SEC-1–10),
                        NIST SSDF (SP 800-218), NIST C-SCRM (SP 800-161r1),
                        SLSA Framework v1.0, EU Cyber Resilience Act (effective 2027)
```

**Overall Risk Rating**: highest severity finding across all phases (Critical > High > Moderate > Low).

**CISA SBOM Compliance**:
- All 8 minimum elements present → Compliant
- 1–3 elements missing → Partial
- No SBOM generated or 4+ elements missing → Non-compliant

---

### Section 2: Management Summary

- Finding count matrix: phases as rows (1–5), severity as columns (Critical / High / Moderate / Low / Total)
- 2–3 sentences on overall supply-chain posture in plain language (no code references)
- Top 5 risks ranked by business impact — one sentence each, no line numbers
- Overall Risk Rating with one-line justification

---

### Section 3: Supply Chain Threat Model

Include an ASCII STRIDE threat map for the supply chain pipeline:

```
  [Registry] ──→ [Lockfile] ──→ [Build Env] ──→ [CI Runner] ──→ [Deploy Artefact]
       │               │               │                │                │
  Spoofing        Tampering        Spoofing         Elevation        Tampering
  Tampering       Info Disc        Tampering        Repudiation      Spoofing
  (typosquat,     (hash swap,      (dep confusion,  (unsigned        (unsigned
   dep confusion)  gitignored)      malicious pkg)    build steps)     artefact)
```

Follow with a one-paragraph narrative of the two or three highest-risk attack paths found in this codebase.

---

### Section 4: SBOM Summary

| Metric | Value |
|--------|-------|
| Total components | N |
| Direct dependencies | N |
| Transitive dependencies | N |
| Unique licenses | list |
| SBOM format generated | CycloneDX vX / SPDX vX |
| CISA minimum elements | N/8 present |
| Missing elements | list or "none" |
| VEX status | N CVEs suppressed / M total CVEs found |
| Provenance coverage | N/M direct deps with signed provenance |
| Grype findings — CI gate (only-fixed) | N critical / N high / N total |
| Grype findings — full (all CVEs) | N critical / N high / N total |
| Grype suppressions (.grype.yaml) | N entries — see Section 5 [Grype-Suppressed] findings |

If multiple ecosystems detected, add per-ecosystem rows for component counts.

**License Risk Sub-table**:

| License | Type | Count | Risk |
|---------|------|-------|------|
| MIT | Permissive | N | Low |
| Apache-2.0 | Permissive | N | Low |
| BSD-2-Clause / BSD-3-Clause | Permissive | N | Low |
| GPL-2.0 | Copyleft | N | Moderate — static-link obligation |
| GPL-3.0 | Copyleft | N | Moderate — review distribution obligation |
| LGPL-2.1 / LGPL-3.0 | Weak copyleft | N | Low — dynamic linking typically ok |
| AGPL-3.0 | Strong copyleft | N | **High** — SaaS loophole; network use triggers obligation |
| UNLICENSED | None | N | Moderate — no clear right to use |
| UNKNOWN | Unresolvable | N | Moderate — requires manual review |

Flag any AGPL-3.0 dependency as High by default in Section 5 findings if the project is a SaaS product.

---

### Section 5: Findings

Group by phase. Within each group, sort Critical → High → Moderate → Low.
Every phase must appear — either with findings or an explicit "no issues found" statement.

**Finding format:**
```
### [Short Title]  [Ecosystem: <name>] (if ecosystem-specific)
**Severity**: Critical / High / Moderate / Low
**CVSS**: <base score e.g. 8.1> (include for all High and Critical CVE findings)
**Attack Pattern**: [e.g. Lifecycle Script Abuse | Lockfile Tampering | Known CVE | Dependency Confusion | Registry Substitution | Typosquatting | Major Version Bump]
**OWASP**: CICD-SEC-X (name) — or N/A
**Package / Location**: <name@version> or <filename:line>
**What happens**: [Observed fact — state what IS true]
**Exploit scenario**: [Required for Critical/High: attacker action → precondition → observable impact]
**Recommendation**: [Specific, actionable fix]
```

**Severity definitions:**
- **Critical**: Active exploit signal; package known-compromised; lockfile completely absent; registry substitution hash mismatch detected
- **High**: CVE CVSS ≥ 7.0; lifecycle script with outbound network call; lockfile integrity failure; floating Docker `FROM` tag; unsigned GitHub Actions on mutable branch ref; AGPL-3.0 in SaaS product
- **Moderate**: CVE CVSS 4.0–6.9; non-registry resolved URL; major version bump without review; new lifecycle script; no SLSA Level 1; dependency confusion risk; Docker tag-pinned (not digest-pinned); UNLICENSED dependency
- **Low**: Best-practice gap; missing SCA in CI; no Dependabot; native binary package; missing VEX infrastructure; no cosign; new package with low publication history

---

### Section 6: PR / Change Checklist

If $ARGUMENTS is a PR number, open with:
```
PR #<n> — <PR title>
Author: <author>
Changed files: <count>
```

Then answer each item — PASS, FAIL, WARN, or N/A with one-line evidence:

| # | Question | Result | Evidence |
|---|----------|--------|----------|
| 1 | Is the lockfile pinned and committed for all detected ecosystems? | | |
| 2 | Did any direct dependency change major version (X.y.z → X+1.y.z)? | | |
| 3 | Were any new packages added? Do they have active maintainers and publication history? | | |
| 4 | Any new lifecycle/install/build scripts in newly added packages? | | |
| 5 | Was SCA (vulnerability scan) run against the updated dependency set? | | |
| 6 | Are builds tested on all required CI targets (OS × arch matrix)? | | |
| 7 | Are any new packages pulling pre-compiled binaries at install time? | | |

Do not hardcode ticket IDs in this table.

---

### Section 7: Remediation Plan

Prioritised by risk. Include retest guidance for every P0 and P1 item.

```
**P0 — Fix before any merge** (Critical/High)
  - [Imperative action title, e.g. "Upgrade lodash to ≥ 4.17.21 (CVE-2021-23337)"]
    Retest: Re-run `npm audit`; confirm 0 critical/high findings; regenerate SBOM and
            confirm integrity hashes present for all components.

**P1 — Fix this sprint** (Moderate with high visibility)
  - [Action title]
    Retest: [Specific command or check to confirm resolution]

**P2 — Improve when possible** (remaining Moderate/Low)
  - [Action title]
```

Use imperative, concise titles throughout.

---

### Appendix A: Framework Mapping

| Finding | OWASP CICD-SEC | NIST SSDF | CISA SBOM | SLSA | EU CRA |
|---------|---------------|-----------|-----------|------|--------|
| Lifecycle script abuse | CICD-SEC-6 | PW.4 | — | L1+ | Art. 13 |
| Dependency confusion | CICD-SEC-6 | PW.4 | — | — | Art. 13 |
| Known CVE unpatched | CICD-SEC-7 | PW.5 | Minimum Elements §3 | — | Art. 13 |
| Lockfile missing/tampered | CICD-SEC-3 | PW.4 | — | L1 | Art. 13 |
| SBOM incomplete | — | PW.4 | Minimum Elements | — | Art. 15 |
| No SLSA provenance | — | PW.4 | — | L1–L3 | Art. 13 |
| Unpinned CI actions | CICD-SEC-6 | PW.4 | — | L2 | Art. 13 |
| No Dependabot/Renovate | CICD-SEC-4 | PW.7 | — | — | Art. 13 |

---

### Appendix B: Revision History

| Version | Date | Change |
|---------|------|--------|
| 1.0 | 2026-05-21 | Initial release — npm/Node.js only |
| 2.0 | 2026-05-27 | Full rewrite: polyglot, ecosystem-agnostic; added VEX, SLSA, typosquatting, license risk, Dockerfile supply chain, EU CRA, CVSS scores, retest guidance |

---

## Completeness Check

Before finalising, verify:
- [ ] Section 0 (Detected Configuration) fully populated
- [ ] Header fully populated — no placeholder values
- [ ] All 5 phases addressed — findings or explicit "no issues found"
- [ ] All Critical/High findings include exploit scenario
- [ ] All High CVE findings include CVSS base score
- [ ] Overall Risk Rating matches highest severity finding
- [ ] CISA compliance verdict matches Section 4 element count
- [ ] Section 6 (PR Checklist) has all 7 rows answered (or N/A if no diff scope)
- [ ] Remediation Plan covers all Critical/High findings with retest guidance
- [ ] No source files were edited, created, or deleted
- [ ] Phase 3.2 run twice — CI-gate and full results both reported in Section 4 and Section 5
- [ ] Phase 3.7 run — .grype.yaml entries listed in Section 5 as [Grype-Suppressed] (or noted as absent)
- [ ] Section 4 Grype finding counts populated for both CI-gate and full scans
- [ ] Section 4 Grype suppression count populated (or "0 — no .grype.yaml present")

## Done When

- [ ] `.md` file written to CWD with correct filename
- [ ] No source files modified
- [ ] No packages installed or removed

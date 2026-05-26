---
name: analyze
description: Run a critical codebase analysis on the current repository
disable-model-invocation: true
user-invocable: true
argument-hint: "[output-directory]"
---

# Critical Codebase Analysis

**Output directory**: $ARGUMENTS (if empty, use the current working directory)

Derive repository metadata from the conversation context (working directory name, git status, recent commits) and today's date. Do NOT run git remote commands. Only use local git commands (e.g. `git rev-parse HEAD`) if the full commit SHA is not already available in the conversation context. Use this metadata to populate the header block and derive the output filename.

**Output filename**: `YYYY-MM-DD-<project-name>-<branch>-CRITICAL-ANALYSIS.md`

---

Critically analyse the entire codebase. Assume production code.
Skip generated files, vendored dependencies, lock files, and build artifacts.

## Ground Rules

- Map the platform, framework, concurrency model, entry points, and
  execution flow before starting.
- Trace execution paths before asserting behaviour. State what DOES happen,
  not what "could" or "might" happen.
- CRITICAL/HIGH findings must include a concrete failure scenario with
  reproduction steps. If you cannot describe one, lower the severity.
- Identify root causes, not symptoms.
- Analyse cross-file interactions — do not review files in isolation.

## Review Areas

### 1. Bugs & Security
- Logic errors, off-by-one, unhandled edge cases
- Race conditions — name threads, shared state, and failure interleaving
- Injection, auth/authz gaps, secrets in code, insecure defaults —
  identify vector and actor
- `.gitignore` audit — flag sensitive files not excluded (`.env`, creds,
  keystores, private keys)
- Tracked secrets — scan git-tracked files for API keys, tokens, passwords,
  or connection strings that should be in environment variables

### 2. Performance
- Algorithmic bottlenecks, N+1 queries, unnecessary allocations
- Memory leaks, missing cleanup/disposal

### 3. Architecture & Design
- Separation of concerns, modularity, SOLID adherence
- Circular dependencies, high coupling, layering violations
- Misused or missing design patterns

### 4. Maintainability
- High cyclomatic complexity, oversized functions/files
- Inconsistent naming, magic values, dead/unused code
- DRY violations and missed abstraction opportunities

### 5. Error Handling & Observability
- Missing/inconsistent error handling — trace catch blocks and their
  actual behaviour
- Silent failures, swallowed errors
- Missing input validation at trust boundaries
- Insufficient logging/observability

### 6. Testing
- Coverage gaps, missing edge-case tests, testability issues
- Flaky tests, over-mocking, under-asserting

### 7. Dependencies & Tech Debt
- Outdated/vulnerable dependencies
- Deprecated API usage
- TODO/FIXME items with risk, temporary workarounds made permanent

### 8. Data Handling & Privacy
- PII logged or exposed in debug output, crash reports, or error messages
- Sensitive data stored unencrypted at rest or transmitted without TLS
- Missing data retention policies, unbounded storage growth
- GDPR/CCPA considerations — right to deletion, consent tracking

### 9. Configuration & Environment
- Hardcoded URLs, ports, credentials, or environment-specific values
- Missing environment variable validation at startup
- Configuration drift risk — values that differ between dev/staging/prod
  without clear documentation
- Feature flags or toggles without expiry or ownership

## Severity Definitions

- **Critical**: Data loss, security breach, or system crash in normal
  operation. No workaround. Requires a specific failure scenario.
- **High**: Production incidents or hard-to-debug bugs in realistic
  scenarios. Requires a concrete example.
- **Medium**: Maintainability or scalability risk.
- **Low**: Clean-up or minor improvement.

If using "could", "might", or "potentially" — that is Medium or below.

## Grading Rubric

Derive the Overall Grading score from finding counts:

| Score | Criteria |
|-------|----------|
| 9–10  | No Critical/High findings. ≤ 3 Medium. Production-ready. |
| 7–8   | No Critical. ≤ 2 High, ≤ 5 Medium. Production-ready with caveats. |
| 5–6   | No Critical. 3+ High or 6+ Medium. Needs work before production. |
| 3–4   | 1+ Critical or 5+ High. Significant risk. Not production-ready. |
| 1–2   | Multiple Critical findings. Fundamental issues. Not production-ready. |

Apply a –1 modifier if findings cluster in Security or Error Handling.
Apply a +1 modifier if strong test coverage mitigates Medium/Low findings.
Final score is clamped to 1–10.

## Issue Format

```
### [Short Title]
**Severity**: Critical / High / Medium / Low
**Location**: file:line
**What happens**: [Actual runtime behaviour]
**Failure scenario**: [precondition → trigger → observable failure]
**Root cause**: [Underlying reason, if different from symptom]
**Recommendation**: [Specific fix or refactor]
```

## Output Structure

1. **Header** — metadata block in this exact format:

```
Repository Name     : <from git remote or folder name>
Branch Name / Commit: <current branch> (<full commit SHA>)
Project             : <from README or package manifest>
Technology Stack    : <primary language, framework, version>
Analysis Date       : <YYYY-MM-DD>
Reviewer            : <from git config user.name or conversation context>
Overall Grading     : <X/10 — derived from findings>
Production Ready    : <Yes / Yes with caveats / No — one-line justification>
```

Source values from repo metadata. Write "Unknown" if undetermined.

2. **Architecture Diagram** — plain-text ASCII box-and-arrow diagram. No Mermaid.
3. **Executive Summary** — overall health, top 5 risks ranked by impact
4. **Strengths** — well-implemented patterns worth preserving
5. **Findings** — grouped by review area, sorted by severity
6. **Strategic Recommendations** — high-impact refactors, architectural
   improvements, testing strategy
7. **Suppressed Findings** — always include this section, even if empty. Format:

```
## Suppressed Findings

These findings were detected but excluded from the grade and Findings section
because they are documented as accepted in the project suppression file.

| # | Severity | Description | Affected Location | Acceptance Reason |
|---|----------|-------------|-------------------|-------------------|
| 1 | Medium   | <what was found> | <file(s)> | <reason from suppression file> |

_If no suppressions were active: write exactly one of these two lines (no path, no filename):_
- `None — no suppression file found.`
- `None — no findings matched the suppression entries.`

**Do NOT include the suppression file path or any reference to 'claude' in this line or anywhere in the report.**
```

The goal is to ensure suppressions are visible and reviewable, not silently hidden.

## Quality Calibration

State facts, not speculation:

- NO: "Security vulnerability"
  YES: "User input at line 20 passed unsanitised to SQL query at line 35,
  allowing injection via the search parameter"

- NO: "Poor performance"
  YES: "O(n²) loop at line 60 processes ~1000 items = 1M iterations,
  adding ~10s latency to the /search endpoint"

Strengths must be equally specific:

- NO: "Good error handling"
  YES: "All Bluetooth callbacks in `BleManager.kt:45-120` use structured
  try/catch with specific exception types and user-facing error messages
  propagated via LiveData"

## Project-Level Suppressions

Before running the analysis, check for a suppression file at `.claude/analyze-suppressions.md`
in the working directory. If it exists, read it before starting the analysis.

Each entry describes a known, accepted issue in plain language. When you encounter a finding
during analysis, use judgment to determine whether it matches a suppression entry — the same
way a human reviewer would recognise "yes, that's the thing we already know about." Do not
rely on title matching; match on the substance of the finding (affected files, the nature of
the issue, the root cause).

For any finding that matches a suppression entry:
- **Omit it from the Findings section entirely**
- **Exclude it from the counts used to derive Overall Grading**
- **Add a "Suppressed Findings" appendix** at the end of the report listing each suppressed
  finding, its severity, and the acceptance reason — so the record stays transparent

If the suppression file does not exist, proceed with no suppressions.

## Completeness Check

Before finalising the report, verify:

- [ ] Every review area (1–9) is addressed — either with findings or an
  explicit "no issues found" statement
- [ ] All Critical/High findings include a failure scenario
- [ ] Overall Grading matches the Grading Rubric criteria
- [ ] Production Ready assessment is consistent with the grading
- [ ] Strengths section contains at least one specific, evidence-based entry
- [ ] Header metadata is fully populated (no placeholder values remaining)
- [ ] Suppression file checked; suppressed findings listed in appendix if any

---
name: pr-review
description: Review a PR between two git branches with full metadata
disable-model-invocation: true
user-invocable: true
argument-hint: "<PR-number-or-URL> OR <source-branch> [target-branch]"
---

# PR Review with Branch Comparison

**Arguments**: $ARGUMENTS

If `$ARGUMENTS` is empty, report an error:
"Usage: /pr-review <PR-number-or-URL> OR /pr-review <source-branch> [target-branch]"

---

## Preflight (run before anything else)

1. **Fetch remote refs** (once only):
   ```bash
   git fetch origin
   ```

2. **Auto-detect repository name**:
   - Use working directory name from conversation context
   - If unavailable, run: `basename $(git rev-parse --show-toplevel)`

3. **Derive output directory**: use the project root:
   ```bash
   git rev-parse --show-toplevel
   ```

4. **Filename normalisation rule**: when building the output filename, replace any `/` in branch names with `-` to avoid creating subdirectories.

---

## Mode Detection

Analyze `$ARGUMENTS` to determine the mode:

**Mode 1: PR Number/URL** (if first argument is purely numeric or a GitHub PR URL matching `/pull/\d+`)
- Extract PR number from argument (either directly or from URL pattern `/pull/(\d+)`)
- This is the RECOMMENDED mode as it auto-detects branches and includes PR metadata
- If the argument looks numeric but `gh pr view` returns an error, fall back to Mode 2 and warn: "Argument looks like a PR number but no PR found — treating as branch name."

**Mode 2: Explicit Branches** (if first argument looks like a branch name)
- First argument: source branch (required)
- Second argument: target branch (optional — auto-detect default branch if not provided)

---

## Instructions by Mode

### Mode 1: PR Number/URL (Recommended)

1. **Check `gh` is available and authenticated**:
   ```bash
   gh auth status
   ```
   If this fails, report: "gh CLI is not installed or not authenticated. Run `gh auth login` then retry." and stop.

2. **Extract PR number**:
   - If argument is numeric: use as-is
   - If argument is URL: extract number from `/pull/(\d+)` pattern

3. **Fetch PR metadata**:
   ```bash
   gh pr view <PR-number> --json headRefName,baseRefName,title,body,labels,url
   ```
   - `headRefName` = source branch
   - `baseRefName` = target branch
   - Capture title, body, labels, url for context

4. **Verify branches exist on remote**:
   ```bash
   git rev-parse --verify origin/<headRefName>
   git rev-parse --verify origin/<baseRefName>
   ```
   If either fails, report: "Branch `<name>` not found on remote. Ensure it has been pushed." and stop.

5. **Capture commit log**:
   ```bash
   git log --oneline origin/<baseRefName>...origin/<headRefName>
   ```

6. **Diff**:
   ```bash
   git diff origin/<baseRefName>...origin/<headRefName>
   ```

7. **Apply review criteria** (see Review Instructions below)

8. **Output filename**: `<project-root>/YYYY-MM-DD-<repo>-<baseRefName>-<headRefName>-PR-REVIEW.md`
   (normalise branch names: replace `/` with `-`)

---

### Mode 2: Explicit Branches

1. **Parse arguments**:
   - Source branch: first argument (required)
   - Target branch: second argument, or auto-detect default branch:
     ```bash
     git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@'
     ```
     Fall back to `main` if detection fails.

2. **Verify branches exist on remote**:
   ```bash
   git rev-parse --verify origin/<source-branch>
   git rev-parse --verify origin/<target-branch>
   ```
   If either fails, report: "Branch `<name>` not found on remote. Ensure it has been pushed." and stop.

3. **Capture commit log**:
   ```bash
   git log --oneline origin/<target-branch>...origin/<source-branch>
   ```

4. **Diff**:
   ```bash
   git diff origin/<target-branch>...origin/<source-branch>
   ```

5. **Apply review criteria** (see Review Instructions below)

6. **Output filename**: `<project-root>/YYYY-MM-DD-<repo>-<target-branch>-<source-branch>-PR-REVIEW.md`
   (normalise branch names: replace `/` with `-`)

---

## Review Instructions (All Modes)

### SETUP:
1. Use remote tracking branches (`origin/...`) to ensure you're reviewing the canonical state
2. Do NOT use local branch versions which may be stale or have uncommitted changes

### PROCESS:
1. For each modified file, read the surrounding context for changed functions and classes — do NOT read the entire file unless it is small (under 150 lines). Rely on diff context for large files.
2. Review ONLY the changed lines, but use surrounding code to assess impact
3. If the diff is very large (100+ files or 3000+ lines), summarise findings by area rather than attempting exhaustive line-by-line review

### SCOPE:
- ONLY report issues in code that appears in the diff
- Do NOT report issues in unchanged code
- If a changed line interacts with unchanged code in a problematic way, that IS in scope
- Trace through call sites and dependencies to assess impact of changes

### GROUND RULES:
- Trace execution paths through the diff before asserting behaviour. State
  what DOES happen, not what "could" or "might" happen.
- For severity requirements, see Severity Definitions below.
- If using "could", "might", or "potentially" — that is Medium or below.

### REVIEW FOR:
- Bugs or logic errors in the changed code
- Security vulnerabilities introduced by the changes (injection, auth gaps,
  secrets, insecure defaults)
- Edge cases not handled in new/modified code
- Breaking changes to existing APIs or contracts
- Race conditions or concurrency issues in changed code
- Resource leaks introduced by the changes
- Error handling gaps in modified code paths
- Data handling issues — PII exposure in logs, unencrypted sensitive data
- Hardcoded configuration — URLs, credentials, environment-specific values
  that should be externalised
- Test coverage — new logic paths, bug fixes, or behavioural changes that
  lack corresponding test additions or updates in the diff. Do not demand
  tests for trivial changes (config, copy, formatting).
- Dependency changes — if the diff modifies a package manifest
  (`package.json`, `build.gradle`, `go.mod`, `requirements.txt`, `Gemfile`,
  `*.csproj`, etc.), flag new or changed dependencies. Check for:
  - Known vulnerabilities (use `gh api` advisories or language-specific
    audit commands where available)
  - Major version bumps that may introduce breaking changes
  - New dependencies that duplicate existing functionality

### SEVERITY DEFINITIONS:
- **Critical**: The change introduces data loss, a security breach, or a crash
  in normal operation. Requires a concrete failure scenario.
- **High**: The change causes production incidents or hard-to-debug failures
  in realistic scenarios. Requires a concrete example.
- **Medium**: The change introduces maintainability or reliability risk that
  will cause problems over time.
- **Low**: Minor improvement — clean-up, naming, or defensive hardening.

### DO NOT IMPLEMENT FIXES:
This skill produces a **review only**. Do not apply any code changes, edits, or fixes
to the repository. Output findings to the review file and stop. Implementation is the
responsibility of the developer who owns the PR.

### DO NOT REPORT:
- Style/formatting preferences
- Pre-existing issues in unchanged code
- "Nice to have" refactoring suggestions
- Documentation gaps (unless the change breaks existing docs)

### QUALITY CALIBRATION:

State facts, not speculation:

- NO: "Potential security issue"
  YES: "User input from `request.query.id` on line 42 is passed unsanitised
  to the SQL query on line 58, allowing injection via the search endpoint"

- NO: "May cause performance issues"
  YES: "New loop at line 30 iterates all users for each request — O(n) per
  call with no caching, will degrade at scale"

### OUTPUT FORMAT:

**Mode 1 — include PR metadata at top:**
```markdown
# PR Review: <PR Title>

**PR Number**: #<number>
**PR URL**: <url>
**Source Branch**: <headRefName>
**Target Branch**: <baseRefName>
**Labels**: <labels>

## PR Description
<body>

## Commits
<git log --oneline output>

---

## Review Findings
```

**Mode 2 — include branch metadata at top:**
```markdown
# PR Review: <repo> — <source-branch> → <target-branch>

**Repository**: <repo>
**Source Branch**: <source-branch>
**Target Branch**: <target-branch>

## Commits
<git log --oneline output>

---

## Review Findings
```

**For each issue found:**

```
### [Short Title]
**Severity**: Critical / High / Medium / Low
**File**: <file>:<line(s)>
**What happens**: [Actual runtime behaviour introduced by this change]
**Failure scenario**: [precondition → trigger → observable failure]
  (required for Critical/High; optional for Medium/Low)
**Suggested fix**: [Specific action]
```

**If no issues found**: "No issues found in the changeset."

**At the end of the review, add a verdict:**

```
## Verdict

**Approve** / **Request Changes** / **Request Changes (blocking)**

[1-2 sentences summarising the overall assessment and what must change
before merge, if anything.]
```

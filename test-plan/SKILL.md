---
name: test-plan
description: Generate a structured manual UI/UX test plan from one or more GitHub PRs
disable-model-invocation: true
user-invocable: true
argument-hint: "<PR-URL-or-number> [PR-URL-or-number ...]"
---

# Manual Test Plan Generator

**Arguments**: $ARGUMENTS

If `$ARGUMENTS` is empty, report an error:
"Usage: /test-plan <PR-URL-or-number> [PR-URL-or-number ...]
Examples:
  /test-plan 372
  /test-plan https://github.com/Org/repo/pull/228
  /test-plan https://github.com/Org/repo/pull/372 https://github.com/Org/repo2/pull/228"

---

## Preflight

1. **Check `gh` is available and authenticated**:
   ```bash
   gh auth status
   ```
   If this fails, report: "gh CLI is not installed or not authenticated. Run `gh auth login` then retry." and stop.

2. **Determine current date and time**:
   ```bash
   date +%Y-%m-%d_%H%M
   ```
   Capture both: date (YYYY-MM-DD) and time (HHmm) for use in the output filename.

3. **Determine output directory**: use the current working directory. If no git repo is present, use `$HOME`.

4. **Validate each PR is accessible**: run `gh pr view` for each PR before proceeding. If any command fails, report which PR could not be accessed (e.g., "PR #372 could not be accessed — check the URL and your repo permissions.") and stop.

---

## Argument Parsing

Parse `$ARGUMENTS` as a space-separated list of one or more PR identifiers.

For each identifier, determine its type:

- **Full GitHub URL** — matches `https://github.com/<owner>/<repo>/pull/<number>`
  - Extract: `owner/repo` (e.g. `Payzone-UK/payzone_app_html5-no-zorin`) and `number` (e.g. `228`)
  - Short repo name for filename: everything after the last `/` in the repo path (e.g. `payzone_app_html5-no-zorin` → `html5`)
    - Simplify to a 2-5 character token: drop common prefixes like `payzone_`, `payzone-`, strip `-no-zorin` suffixes. Use the most distinctive part.
  - Use `--repo <owner>/<repo>` flag in all `gh` commands

- **Numeric string** — treat as a PR number in the **current repository**
  - Auto-detect current repo: `basename $(git rev-parse --show-toplevel)` — fall back to "current-repo" if not in a git repo
  - No `--repo` flag needed

If any identifier cannot be parsed as either form, report: "Cannot parse argument '<arg>' — expected a PR number or a GitHub PR URL." and stop.

---

## Data Collection

Collect data for ALL PRs before generating any output.

For each PR:

### 1. Fetch PR metadata

```bash
# For URL-based (with --repo):
gh pr view <number> --repo <owner>/<repo> \
  --json number,title,body,headRefName,baseRefName,labels,url,additions,deletions,files

# For number-only (current repo):
gh pr view <number> \
  --json number,title,body,headRefName,baseRefName,labels,url,additions,deletions,files
```

Capture: number, title, body (PR description), headRefName, baseRefName, labels, url, additions, deletions, files (list of changed filenames).

### 2. Fetch diff

```bash
# For URL-based:
gh pr diff <number> --repo <owner>/<repo>

# For number-only:
gh pr diff <number>
```

**Important**: fetch the full diff **once** and analyse it from the output. Do not run additional per-file extraction commands (separate awk or grep calls against a second `gh pr diff` invocation) — read all changed files from the single diff output.

### 3. Classify changed files

From the diff, categorise changed files into:
- **UI Templates**: `*.html`, `*.erb`, `*.slim`, `*.haml`, `*.hbs`, `*.jinja`, `*.liquid`, `*.mustache`
- **Components**: `*.vue`, `*.jsx`, `*.tsx`, `*.svelte`, `*.angular.html`
- **Stylesheets**: `*.css`, `*.scss`, `*.less`, `*.sass`
- **Client JS/TS**: `*.js`, `*.ts` — exclude `*.test.*`, `*.spec.*`, `*.min.*`
- **Routes/Config**: files named `routes.rb`, `router.js`, `routes.js`, `*.config.*`, navigation-related files
- **Backend/Model**: everything else (Ruby models, controllers, Python views, Go handlers, etc.)

For large diffs (50+ files), read the diff content for UI Templates, Components, and Stylesheets in full. For Backend/Model files, read only the file names and any context from the PR description — do not attempt to read every line.

---

## Analysis

Before writing any output, perform this analysis across all collected PR data:

### Change areas
Group related changes into named areas. Each area = one coherent set of changes affecting the same feature, page, or concern. Examples: "Desktop Layout — Financials Page", "Mobile Layout", "Sass Build Fixes", "Receipt Routing Logic", "Search Filter Hardening".

One area may span multiple files. One file may contribute to multiple areas only if the changes are genuinely distinct.

### Visual/functional impact
For each area, determine:
- What the user saw/experienced **before** the change
- What the user sees/experiences **after** the change
- Whether the change is visible to end users, or invisible (build fix, internal logic)

### Risk assessment
Derive an overall Risk Level:
- **High**: Changes affect transactional flows, auth, data integrity, or navigation
- **Medium**: Layout changes on primary user-facing pages, new UI states (error, loading, empty), search/filter behaviour
- **Low**: Cosmetic/spacing fixes, build-only changes, minor copy updates, syntax fixes with no runtime visual change

When writing the Risk Level justification in the output, always state *why* — name the specific types of code that were **not** touched (e.g., "no transactional, auth, or data handling code was modified") as well as what was. A one-word risk label without rationale is not acceptable.

### Test case derivation
For each change area, derive 1-3 test cases. Each test case must:
- Be executable by a non-technical manual tester (no code, no CLI, no dev tools required)
- Cover the primary happy path AND at least one error/edge case for High-priority areas
- Describe user actions ("Navigate to...", "Click...", "Enter...", "Resize the browser to portrait orientation")
- State a specific, observable expected result

**Priority assignment rules**:
- If the change affects the critical user path (checkout, payment, login, data entry forms): **High** — test these first regardless of change size
- If visual regression on the changed page could block a release: **Medium**
- If the change is a build fix, infrastructure upgrade, or cosmetic tweak with no user-facing logic: **Low** (one smoke test is sufficient)

---

## Output

### Filename
`YYYY-MM-DD-HHmm-<repo-tokens>-TEST-PLAN.md`

- Single PR: `2026-05-08-1430-html5-TEST-PLAN.md`
- Multiple PRs from different repos: join short tokens with `-`: `2026-05-13-0915-html5-models-TEST-PLAN.md`
- Multiple PRs from same repo: use repo token once

Write to the current working directory (project root if in a git repo).

---

### Document content

Write the following sections in order:

---

#### SECTION 1 — Document Header

```markdown
# YYYY-MM-DD — [Repo Short Name(s)] — [PR Title or brief combined description] — Test Plan

**Generated**: YYYY-MM-DD
**Source PR(s)**: [#228 — repo name](url) · [#372 — repo name](url)
**Scope**: Manual UI/UX Testing
**Risk Level**: Low / Medium / High
```

---

#### SECTION 2 — Executive Summary

One paragraph only. Plain English — no file paths, no variable names, no code references.

Describe what changed from a **user's perspective** and why it matters for testing. Source from: PR description(s) → diff analysis → inferred user impact. If PR description is empty, derive entirely from diff.

> Example: "This test plan covers CSS modernisation changes to the Financials Admin Page, plus Sass syntax fixes required for Node 24 compatibility. The layout has been updated for improved centring and responsive behaviour on both desktop and mobile. Manual testing should confirm visual alignment on both orientations and that the page loads without errors."

---

#### SECTION 3 — Key Changes

One numbered subsection per change area identified in the analysis.

Format for each area:

```markdown
### N. <Area Name>

**File(s)**: `path/to/file.ext`

| Change | Before | After |
|--------|--------|-------|
| [What changed — written as a user-understandable description, not code] | [How it appeared or behaved before — be specific] | **[How it appears or behaves now — bold the key improvement]** |
| ... | ... | ... |

> **Visual Result:** [One sentence describing the net visual or functional outcome for the user.]
```

For areas with no visible user impact (e.g., build fixes, syntax corrections):
- Before column: "Page failed to load / build produced errors"
- After column: "Page loads correctly" / "Build completes without errors"
- Visual Result: "No visual change to the rendered page, but the application now builds and runs correctly."

For backend/model changes with indirect UI impact:
- Describe the impact in terms of what the user sees in the UI, not what the server does internally
- E.g., "Search results no longer return unexpected records when special characters are entered"

---

#### SECTION 4 — Scope & Risk

```markdown
## Scope & Risk

| Aspect | Details |
|--------|---------|
| **Scope** | [What pages/features are directly affected. What is explicitly NOT affected.] |
| **Breaking Changes** | None — [describe rationale] / [describe breaking change if any] |
| **Browser / Device Support** | [Which browsers and devices are relevant to this change] |
| **Risk Level** | Low / Medium / High — [Justify by naming what was changed AND what was not. E.g., "Low — layout refactoring only; no transactional, auth, or data handling code was modified. Changes are isolated to specific pages."] |
| **Testing Required** | [Type of testing: visual regression, functional, regression spot-check, etc.] |
```

---

#### SECTION 5 — Bottom Line

Three to five bullet points. One per major change area. Plain English. Each bullet should name the specific thing to verify — not just that something changed.

```markdown
## Bottom Line

- ✅ **[Area Name]** — [One sentence: what changed and exactly what to verify — be specific about the observable outcome.]
- ✅ **[Area Name]** — [One sentence.]
- ✅ **[Area Name]** — [One sentence.]
```

> Bad: "Layout refactored. Verify it looks correct."
> Good: "Layout refactored. Verify the X-Total title and value are both centred, buttons are 80% width with equal gaps, and the 'Reset Transactions (Z Total)' label wraps cleanly across two lines."

---

#### SECTION 6 — Test Cases

```markdown
## Test Cases

| ID | Area | Priority | Manual Steps | Expected Result | Pass | Fail | Tested By | Build |
|----|------|----------|--------------|-----------------|------|------|-----------|-------|
| MT-01 | [Area name] | High | 1. ...<br>2. ...<br>3. ... | [What the tester should observe] | [ ] | [ ] | | |
```

**Priority values**: `High` / `Medium` / `Low`

**MT-ID numbering**: sequential from MT-01 across all PRs if multiple sources.

**Writing manual steps**:
- Number each step
- Write as user actions only: "Navigate to...", "Click the [button name] button", "Enter [value] in the [field name] field", "Resize the browser window to a narrow (mobile) width"
- No code, no class names, no variable names, no developer tools
- Max 5 steps per test case — split into multiple cases if more are needed
- Happy path cases first, then edge/error cases

**Writing expected results**:
- Describe what the tester should **see** or **observe**
- Be specific: "The X-Total title and value should appear centred both horizontally and vertically on screen" is good; "The layout should look correct" is not

**Priority guidance**:
- **High**: Transactional flows, auth, security-sensitive behaviour, data correctness, navigation, critical user path (checkout, payment, login) — test these first
- **Medium**: Layout on primary pages, new UI states (error messages, loading indicators, empty states), filter/search behaviour
- **Low**: Cosmetic fixes, spacing, build-only changes — verify basic page load and visual sanity

---

#### SECTION 7 — Regression Areas (include only if applicable)

Only include this section if the diff touches: shared navigation, authentication, widely-used UI components, shared stylesheets affecting multiple pages, or build tooling that could affect all pages.

```markdown
## Regression Areas

The following areas were not directly changed but may be affected. Perform a brief spot-check:

- **[Area name]**: [One sentence on why it is at risk. Name specific pages or journeys to check where possible — avoid generic references like "all pages using shared styles".]
```

Omit this section entirely if there are no meaningful regression risks.

---

#### SECTION 8 — Out of Scope

```markdown
## Out of Scope

The following are not covered by this test plan:

- Performance and load testing
- Automated test suite execution
- Backend API contract testing
- Database integrity verification
- Accessibility audit (flag for separate review if significant UI changes were made)
```

---

## Quality Rules

- **No code in the output** — no variable names, class names, CSS property names, file paths (except in Section 3 file references), or terminal commands. The document must be readable by a non-technical QA tester.
- **State facts, not speculation** — describe what the change does, not what it "might" do.
- **Before/After must be specific** — "Left-aligned, not vertically centered" is good; "looked wrong" is not.
- **Test steps must be actionable** — if a step requires developer access or knowledge to execute, rewrite it.
- **Do not implement fixes or suggest code changes** — this skill produces a test plan only.
- **Risk Level must be justified** — always explain what types of code were and were not touched. A bare "Low" / "Medium" / "High" label with no rationale is not acceptable.
- **For multi-PR plans**: synthesize into a coherent document; do not simply concatenate one plan per PR.
  - Group related changes across PRs (e.g., "Authentication — Backend + Frontend")
  - If two PRs change the same file or area, merge them into one Key Changes entry
  - Note dependencies where they exist: "PR #372 adds the data model; PR #228 adds the UI"
  - In test cases, note which PRs a test covers when it spans multiple: add `[PR #372, #228]` in the Area column

---

## DO NOT IMPLEMENT

This skill produces a **test plan document only**. Do not apply any code changes, edits, or fixes to the repository.

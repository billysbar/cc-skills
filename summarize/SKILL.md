---
name: summarize
description: Produce a management-facing summary from a CRITICAL-ANALYSIS.md file
disable-model-invocation: true
user-invocable: true
argument-hint: "<path-to-CRITICAL-ANALYSIS.md>"
---

# Management Summary from Critical Analysis

**Input file**: $ARGUMENTS

Read the CRITICAL-ANALYSIS.md file at the path above. Derive the output filename from the input filename by replacing `CRITICAL-ANALYSIS.md` with `SUMMARY-ANALYSIS.md`. Write the output alongside the input file.

If `$ARGUMENTS` is empty, report an error: "Usage: /summarize <path-to-CRITICAL-ANALYSIS.md>"

After reading the file, verify it contains the expected structure:
- A metadata header block (Repository Name, Overall Grading, etc.)
- At least one severity-tagged finding (Critical/High/Medium/Low)

If either is missing, report an error: "The input file does not appear to be
a valid CRITICAL-ANALYSIS.md — missing [header block / severity-tagged findings].
Please check the file path."

---

Review the provided CRITICAL-ANALYSIS.md file and produce a management-facing
summary report. Work ONLY from the CRITICAL-ANALYSIS.md content. Do not access,
reference, or infer anything from the original codebase.

If the CRITICAL-ANALYSIS.md lacks sufficient detail to produce a meaningful
summary — e.g. missing severity ratings, no failure scenarios, or vague
findings — state this explicitly at the top of the output and recommend
reviewing the CRITICAL-ANALYSIS.md directly. Do not fabricate or infer
information that is not present.

## Audience

Management and non-technical stakeholders who will create JIRA tickets and
assign remediation work to engineers. The summary must be:

- Succinct — no code snippets, no line-level detail
- Actionable — each finding maps to a discrete work item
- Prioritised — highest impact first
- Jargon-light — explain technical concepts in plain terms where possible

## Severity Filter

Include only **Medium** severity and above (Critical, High, Medium).
Omit all Low severity findings entirely.

## Output Structure

### 1. Header

Copy the metadata block verbatim from the CRITICAL-ANALYSIS.md:

```
Repository Name     : ...
Branch Name / Commit: ...
Project             : ...
Technology Stack    : ...
Analysis Date       : ...
Reviewer            : ...
Overall Grading     : ...
Production Ready    : ...
```

Below the header block, add a one-line grading interpretation:
> **Grading Context**: [X]/10 — [rubric band description from the analysis]

### 2. Architecture Diagram

If this summary is the primary document stakeholders will receive, copy the
ASCII architecture diagram verbatim from the CRITICAL-ANALYSIS.md. If
stakeholders also receive the full CRITICAL-ANALYSIS.md, omit the diagram
and add: "See Architecture Diagram in CRITICAL-ANALYSIS.md".

### 3. Executive Summary

Open with a finding count table:

| Severity | Count |
|----------|-------|
| Critical |   X   |
| High     |   X   |
| Medium   |   X   |

Then:
- 2-3 sentences on overall codebase health
- A numbered list of the **top 5 risks**, ranked by impact
- Each risk: one sentence stating what is wrong and why it matters

### 4. Key Strengths (max 3 items)

Include up to 3 strengths from the CRITICAL-ANALYSIS.md, rewritten in
plain language. This provides stakeholders with a balanced view.
If the CRITICAL-ANALYSIS.md has no strengths section, omit this section.

### 5. Findings

Group findings by severity tier, not by review area:

#### Critical Findings
#### High Findings
#### Medium Findings

For each finding use this format:

```
### [Short Title]
**Severity**: Critical / High / Medium
**Category**: Bugs & Security / Performance / Architecture & Design /
              Maintainability / Error Handling & Observability / Testing /
              Dependencies & Tech Debt / Data Handling & Privacy /
              Configuration & Environment
**Impact**: [Plain-language description of what goes wrong and who is affected —
            no code, no line numbers]
**Recommendation**: [What to do about it — action, not implementation detail]
```

- Do not include code snippets or file:line references — engineers can
  cross-reference the CRITICAL-ANALYSIS.md for specifics
- Preserve the failure scenario from the CRITICAL-ANALYSIS.md but rewrite it
  in plain language (cause → effect → business impact)
- If the CRITICAL-ANALYSIS.md finding lacks a clear failure scenario, state
  that the full detail is in the CRITICAL-ANALYSIS.md

### 6. Prioritised Remediation Plan

Organise actionable items into priority tiers:

| Priority | Description | Criteria |
|----------|-------------|----------|
| **P0 — Immediate** | Must fix before any deployment | Critical severity, security breaches, data loss |
| **P1 — Next Sprint** | Fix before production release | High severity, production incident risk |
| **P2 — Short-Term** | Schedule within current quarter | Medium severity, maintainability / scalability risk |

Use a table per priority tier:

```
| # | Action | Severity | Category | Effort |
|---|--------|----------|----------|--------|
```

Estimate effort as S (hours), M (days), or L (sprint+). Derive from finding
complexity — single-file fixes are S, cross-cutting changes are L.

Each action should be written as a JIRA-ready title (imperative verb,
specific scope, e.g. "Rotate committed API credentials and purge from
git history").

### 7. Conclusion

- **Production Readiness**: [Copy verdict from header] — [one sentence
  on minimum effort to reach deployment, if applicable]
- **Recommendation**: Deploy / Deploy with caveats / Do not deploy

## Quality Rules

- Every finding in the summary must trace back to a finding in the
  CRITICAL-ANALYSIS.md — do not add new findings
- Do not upgrade or downgrade severity ratings from the CRITICAL-ANALYSIS.md
- Strengths are limited to 3 items maximum and must come from the
  CRITICAL-ANALYSIS.md — do not invent positive observations
- Keep the total output concise — aim for roughly 30-50% of the
  CRITICAL-ANALYSIS.md length
- If findings in the CRITICAL-ANALYSIS.md carry source or tool attribution
  tags, preserve them as-is in the summary

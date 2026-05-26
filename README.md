# Claude Code Skills

User-defined skills for Claude Code. Each subdirectory contains a `SKILL.md` that defines the skill's behaviour.

## Skills

| Skill | Description |
|---|---|
| `analyze` | Run a critical codebase analysis on the current repository |
| `jira-ticket` | Fetch a Jira ticket by ID and write it to `{TICKET-ID}.md` in the current directory |
| `pr-review` | Review a PR between two git branches with full metadata |
| `sbom-audit` | Run an SBOM and supply-chain security audit on an npm/Node.js project |
| `security-audit` | Run a CSWG-aligned security audit on the current codebase |
| `self` | List all user-defined skills with usage summaries |
| `summarize` | Produce a management-facing summary from a `CRITICAL-ANALYSIS.md` file |
| `test-plan` | Generate a structured manual UI/UX test plan from one or more GitHub PRs |

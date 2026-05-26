---
name: self
description: List all user-defined skills with usage summaries
disable-model-invocation: true
user-invocable: true
---

# Skill Directory

List every skill installed under `~/.claude/skills/` by reading the `SKILL.md` frontmatter in each subdirectory.

## Steps

1. Run the following to collect skill metadata:
   ```bash
   for skill in ~/.claude/skills/*/SKILL.md; do
     echo "=== $skill ==="
     head -20 "$skill"
     echo ""
   done
   ```

2. For each skill directory found, extract from the frontmatter:
   - `name` — the slash command (e.g. `/name`)
   - `description` — one-line summary of what it does
   - `argument-hint` — usage hint (omit if not present)

3. Print the results as a markdown table with columns: **Skill**, **Usage**, **Description**.
   Format the skill name as a slash command (e.g. `/jira-ticket`).
   If `argument-hint` is present, show it in the Usage column as `/name <hint>`.
   If not, just show `/name`.

   Example output:
   ```
   | Skill | Usage | Description |
   |---|---|---|
   | `/jira-ticket` | `/jira-ticket <TICKET-ID or full URL>` | Fetch a Jira ticket and write it to {TICKET-ID}.md |
   ```

4. After the table, print a one-line footer:
   ```
   {n} skills installed  ·  ~/.claude/skills/
   ```

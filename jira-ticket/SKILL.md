---
name: jira-ticket
description: Fetch a Jira ticket by ID and write it to {TICKET-ID}.md in the current directory
disable-model-invocation: true
user-invocable: true
argument-hint: "<TICKET-ID or full URL>  e.g. PROJ-123 or https://company.atlassian.net/browse/PROJ-123"
---

# Jira Ticket → Markdown

**Arguments**: $ARGUMENTS

---

## Step 1: Validate input

If `$ARGUMENTS` is empty, stop and print:
```
Usage: /jira-ticket <TICKET-ID>
Example: /jira-ticket PROJ-123
```

If `$ARGUMENTS` looks like a full Jira URL (e.g. `https://company.atlassian.net/browse/PROJ-123`):
- Extract the ticket ID from the last path segment: `PROJ-123`
- Extract the base URL: `https://company.atlassian.net` — use this as `JIRA_BASE_URL` even if the env var is unset (URL takes precedence)

Otherwise treat `$ARGUMENTS` as a bare ticket ID.

Normalise the ticket ID to uppercase (e.g. `proj-123` → `PROJ-123`).

---

## Step 2: Check required environment variables

Run:
```bash
echo "JIRA_BASE_URL=${JIRA_BASE_URL}" && echo "JIRA_EMAIL=${JIRA_EMAIL}" && echo "JIRA_API_TOKEN=${JIRA_API_TOKEN:+set}"
```

If any of `JIRA_BASE_URL`, `JIRA_EMAIL`, or `JIRA_API_TOKEN` is unset or empty, stop and print this setup message exactly:

```
Missing Jira credentials. Add these to your ~/.zshrc:

  export JIRA_BASE_URL="https://yourcompany.atlassian.net"
  export JIRA_EMAIL="you@example.com"
  export JIRA_API_TOKEN="your-token"

Generate an API token at: https://id.atlassian.com/manage-profile/security/api-tokens

Then run: source ~/.zshrc
```

---

## Step 3: Fetch the ticket

Run this curl command (v2 API — plain text description):
```bash
curl -s -f -u "${JIRA_EMAIL}:${JIRA_API_TOKEN}" \
  "${JIRA_BASE_URL}/rest/api/2/issue/${TICKET_ID}?fields=summary,description,issuetype,status,priority,assignee,reporter,labels,components,fixVersions,subtasks,comment,parent,created,updated,attachment"
```

where `${TICKET_ID}` is the normalised argument from Step 1.

If curl returns a non-zero exit code or the JSON contains an `errorMessages` or `errors` key, stop and print the error. Common cases:
- HTTP 401: credentials are wrong or the API token has been revoked
- HTTP 404: ticket does not exist or the user does not have permission to view it

---

## Step 4: Parse fields with jq

Use `jq` to extract the following from the JSON response. Treat any `null` value as an empty string or omit the row:

- `.fields.summary`
- `.fields.issuetype.name`
- `.fields.status.name`
- `.fields.priority.name`
- `.fields.assignee.displayName` (null → "Unassigned")
- `.fields.reporter.displayName`
- `.fields.labels | join(", ")` (empty array → omit row)
- `.fields.components | map(.name) | join(", ")` (empty → omit row)
- `.fields.fixVersions | map(.name) | join(", ")` (empty → omit row)
- `.fields.parent.key` + `.fields.parent.fields.summary` (omit section if null)
- `.fields.created | split("T")[0]`
- `.fields.updated | split("T")[0]`
- `.fields.description` (may be null — render as "(no description)" if so)
- `.fields.subtasks[]` — for each: `.key`, `.fields.summary`, `.fields.status.name`
- `.fields.comment.comments[-5:][]` — last 5 comments; for each: `.author.displayName`, `.created | split("T")[0]`, `.body`
- `.fields.attachment[]` — for each: `.filename`, `.content` (download URL), `.mimeType`, `.size`

---

## Step 5: Download attachments

Determine the image MIME types: `image/png`, `image/jpeg`, `image/gif`, `image/webp`, `image/svg+xml`.

If `.fields.attachment` contains any entries:

1. Create a directory: `{CWD}/{TICKET-ID}-attachments/`
2. For **each attachment** (images and non-images alike), download it with curl:
   ```bash
   curl -s -L -u "${JIRA_EMAIL}:${JIRA_API_TOKEN}" \
     "{attachment.content}" \
     -o "{TICKET-ID}-attachments/{attachment.filename}"
   ```
3. For **image attachments**, base64-encode the downloaded file using Python and store the data URI for embedding in Step 6:
   ```bash
   python3 -c "
   import base64
   with open('{TICKET-ID}-attachments/{filename}', 'rb') as f:
       b64 = base64.b64encode(f.read()).decode('ascii')
   print(f'data:{mimeType};base64,{b64}')
   "
   ```
4. Keep a list of:
   - **image attachments**: those whose `mimeType` starts with `image/` — embed inline as base64 data URIs
   - **other attachments**: everything else — filename + size (format size as KB if ≥ 1024 bytes, else bytes)

If there are no attachments at all, skip this step entirely (do not create the directory).

---

## Step 6: Write the output file

Output path: `{CWD}/{TICKET-ID}.md`  (e.g. `./PROJ-123.md`)

Format the file exactly as shown below. Omit any table row or section whose value is empty/null.

```markdown
# {TICKET-ID}: {summary}

| Field | Value |
|---|---|
| Type | {issuetype} |
| Status | {status} |
| Priority | {priority} |
| Assignee | {assignee} |
| Reporter | {reporter} |
| Labels | {labels} |
| Components | {components} |
| Fix Version | {fixVersions} |
| Parent | {parent.key}: {parent.summary} |
| Created | {created} |
| Updated | {updated} |

---

## Description

{description}

---

## Subtasks

- [ ] {subtask.key} · {subtask.status} · {subtask.summary}

---

## Screenshots & Images

![{filename}]({data-uri-for-filename})

---

## Attachments

- {filename} ({size})

---

## Comments

### {author} — {date}

{body}

---
```

Rules:
- Each comment gets its own `### Author — Date` heading followed by the body, then a `---` divider
- If there are no subtasks, omit the Subtasks section entirely
- If there are no comments, omit the Comments section entirely
- If there are no image attachments, omit the Screenshots & Images section entirely
- If there are no non-image attachments, omit the Attachments section entirely
- If there are no attachments at all, omit both sections
- Preserve line breaks in description and comment bodies
- Images appear in the order they were attached (as returned by the API)

---

## Step 7: Report result

Print a summary, for example:
```
Written: ./PROJ-123.md
Attachments: 3 downloaded to ./PROJ-123-attachments/ (2 images, 1 other)
```
If there were no attachments, just print:
```
Written: ./PROJ-123.md
```

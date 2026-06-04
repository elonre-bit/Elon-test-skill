---
name: publishers-support-auto-investigate
description: >-
  Automated publisher support investigation triggered by a Slack message in #publishers-support-cases.
  Extracts the Zendesk ticket ID from the Slack message, runs the full support playbook investigation
  (classify → playbooks → expected behavior → past resolutions → R&D cases), and posts the findings
  as a reply in the same Slack thread. Use when the Publishers Support — Auto Investigate automation
  fires, or when asked to investigate a Slack support notification automatically.
---

# Publishers Support — Auto Investigate

Automated skill for the **Publishers Support — Auto Investigate -Elon** Cursor Automation.

---

## Step 1 — Extract Zendesk ticket ID from the Slack message

The triggering message comes from the **Appcharge Support** Slack bot (`U07CXNMD0Q7`) and uses Slack Block Kit. The plain-text body may be empty; the ticket link is embedded in the blocks/attachments.

Use this priority order to find the ticket ID:

1. **URL pattern** in message text or attachment URLs:
   ```
   https://appcharge.zendesk.com/agent/tickets/(\d+)
   ```
2. **Hash pattern** in message text:
   ```
   #(\d+)
   ```
3. **Fallback** — read the thread with `slack_read_thread` to check if a later message in the thread contains the ticket ID.

If no ticket ID is found after all three attempts, post a reply to the thread:
> ⚠️ Could not extract a Zendesk ticket ID from this message. Please investigate manually.

Then stop.

---

## Step 2 — Fetch the Zendesk ticket

```
tool: zendesk_get_ticket_details (user-zendesk)
ticket_id: <extracted ID>
```

Extract from the ticket:
- Subject
- Publisher Name (custom field `25474794377117`)
- Topic category (custom field `25439025195037` — value starts with `b2b_ticket_topic_*`)
- Environment (custom field `28475093715869`: `production`, `sandbox`, `production_sandbox`)
- Player ID, Order ID, Session ID (from ticket body if present)

**Fast-exit cases — handle before investigation:**
- **Operational request** (e.g. "add user to dashboard", "update config", "grant access"): skip investigation and reply with the action needed.
- **Single-player transient report** (one player, no order ID, issue not reproducible): reply suggesting basic troubleshooting first (clear cache, different browser, reopen store).

---

## Steps 3–6 — Playbook Investigation

Follow the same phases as the `support-playbook-advisor` skill exactly:

**Phase 0 — Classify**
Classify into: `Dashboard` | `Checkout/Payments` | `Store/Offers` | `Award` | `SDK Events` | `Mobile` | `API/Reporting` | `Auth` | `Other`

**Phase 1 — Playbooks**
- Read `PLAYBOOKS.md` from the support-playbook-advisor skill directory at `/Users/elonreiner/.cursor/skills/support-playbook-advisor/PLAYBOOKS.md`
- Search Confluence:
  ```
  tool: searchConfluenceUsingCql (user-atlassian-mcp)
  cql: space.key = "Support" AND (text ~ "[keyword1]" OR text ~ "[keyword2]") AND type = page
  ```
- If Publisher Name found, also search:
  ```
  cql: space.key = "Support" AND title = "[PublisherName]" AND type = page
  ```
- For each relevant Confluence page, call `getConfluencePage` with `contentFormat: "markdown"` before writing output.

**Phase 2 — Expected behavior?**
- Fetch the relevant `docs.appcharge.com` page for the classified flow.
- If 404: fetch `https://docs.appcharge.com/llms.txt` to find the correct path.

**Phase 3 — Past resolutions (run in parallel)**
- 3a: Search Confluence Support space for past resolution write-ups.
- 3b: Search Jira for open ACDEV tickets:
  ```
  tool: searchJiraIssuesUsingJql (user-atlassian-mcp)
  cloudId: 38515cfd-c8c7-4d33-80dc-c473e0ae2550
  jql: project = ACDEV AND text ~ "[symptom keywords]" AND created >= -270d ORDER BY created DESC
  ```

---

## Step 7 — Build the reply message

Format the output using this exact template (standard markdown):

```
## Ticket #[ID] — [Subject]

**Publisher:** [Name] · **Environment:** [Production / Sandbox] · **Opened:** [date]
> [One-sentence plain-English summary]

---

### Classification
**`[Category]`** — [one-line description]

---

### Verdict
| | |
|---|---|
| Expected behavior? | ✅ Yes / ❌ No |
| Appcharge at fault? | ✅ Yes / ❌ No / ⚠️ Unknown — [reason] |
| Playbook | ✅ [Name](link) / ❌ None found |

---

### Past Resolutions
_(Omit if nothing found)_
- [How a similar issue was resolved]

---

### Known R&D Cases
_(Omit if nothing found)_
- **[ACDEV-XXX](link)** `Open` — [description]

---

### Next Steps
1. [First concrete action]
2. [Second action]
```

If `Playbook = None found` AND both Past Resolutions and Known R&D Cases are empty, append:
> ⚠️ **No playbook and no known case exist for this issue type.** If this ticket gets resolved, consider documenting the solution in PLAYBOOKS.md.

---

## Step 8 — Post reply to Slack thread

```
tool: slack_send_message (plugin-slack-slack)
channel_id: <channel ID from the trigger>
message: <formatted output from Step 7>
thread_ts: <message_ts of the triggering message>
```

Do **not** set `reply_broadcast: true` — keep the reply in-thread only.

---

## Guardrails

- **READ-ONLY on Zendesk and Confluence**: never modify, comment on, or close Zendesk tickets; never create or edit Confluence pages.
- Only run Coralogix logs when the user explicitly requests log investigation.
- Do not post more than one Slack reply per invocation.

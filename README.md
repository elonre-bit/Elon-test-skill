# Publishers Support — Auto Investigate

A **Cursor Agent Skill** that automates first-response investigation for publisher support tickets. When a new support case lands in Slack, this skill kicks in, fetches the ticket from Zendesk, runs a full playbook investigation, and posts a structured findings summary back into the same Slack thread — all without any manual intervention.

---

## How It Works

| Step | What happens |
|------|--------------|
| **1. Trigger** | A message arrives in `#publishers-support-cases` from the Appcharge Support bot |
| **2. Extract** | The skill parses the Slack message to find the Zendesk ticket ID |
| **3. Fetch** | Pulls full ticket details from Zendesk (subject, publisher, environment, IDs) |
| **4. Classify** | Categorises the issue: Dashboard / Checkout / Store / Award / SDK / Auth / etc. |
| **5. Investigate** | Runs a 3-phase playbook check: playbooks → expected behavior → past resolutions & R&D cases |
| **6. Reply** | Posts a structured Markdown summary back to the triggering Slack thread |

---

## Output Format

Every reply follows a fixed template:

```
## Ticket #[ID] — [Subject]

Publisher · Environment · Opened date
> One-sentence summary

### Classification
### Verdict  (expected behavior? / at fault? / playbook?)
### Past Resolutions
### Known R&D Cases
### Next Steps
```

---

## Integrations Used

| Integration | Purpose |
|-------------|---------|
| **Slack** (`plugin-slack-slack`) | Read triggering message, post reply to thread |
| **Zendesk** (`user-zendesk`) | Fetch ticket details |
| **Confluence** (`user-atlassian-mcp`) | Search playbooks and past resolutions |
| **Jira** (`user-atlassian-mcp`) | Search for open ACDEV R&D tickets |
| **docs.appcharge.com** | Verify expected product behavior |

---

## Guardrails

- Read-only on Zendesk and Confluence — never modifies or closes tickets, never edits pages.
- Posts exactly one Slack reply per invocation.
- Does not run Coralogix log investigation unless explicitly requested.
- Fast-exits for operational requests (e.g. "add user") and single-player transient reports.

---

## Setup

This skill is designed to be used with the **Publishers Support — Auto Investigate -Elon** Cursor Automation, which is triggered by a Slack message in `#publishers-support-cases`.

Required connected MCP servers in Cursor:
- `plugin-slack-slack`
- `user-zendesk`
- `user-atlassian-mcp`

For full setup instructions, refer to the [Cursor Automations documentation](https://cursor.com/docs).

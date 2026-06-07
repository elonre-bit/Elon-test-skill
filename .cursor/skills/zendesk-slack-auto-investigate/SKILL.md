---
name: zendesk-slack-auto-investigate
description: >-
  Automated support investigation triggered by a Zendesk bot message in Slack.
  Extracts the ticket ID from the Slack message, fetches ticket details via Zendesk MCP,
  runs the full playbook investigation (classify → playbooks → expected behavior → past resolutions → known R&D cases),
  and replies in the same Slack thread with a structured verdict.
  Use when a Zendesk bot posts a new ticket notification in a Slack channel and the automation fires.
---

# Zendesk–Slack Auto-Investigate

Automated skill triggered by a **Zendesk bot message in Slack**.

---

## MCP Tools Required

| Server | Tools used |
|---|---|
| `plugin-slack-slack` | `slack_read_thread`, `slack_send_message` |
| `user-zendesk` | `zendesk_get_ticket`, `zendesk_get_ticket_details` |
| `user-atlassian-mcp` | `searchConfluenceUsingCql`, `getConfluencePage`, `searchJiraIssuesUsingJql`, `getJiraIssue` |
| `plugin-atlassian-atlassian` | `searchConfluenceUsingCql`, `searchJiraIssuesUsingJql` (alternative) |

---

## Step 1 — Extract the Zendesk ticket ID

### Understanding the Zendesk bot message format

The triggering message comes from the **Appcharge Support** Slack bot (`U07CXNMD0Q7`).
This bot uses **Slack Block Kit** to format its messages. The Slack MCP tools (`slack_read_channel`, `slack_read_thread`, `slack_search_public`) **do NOT surface Block Kit content** — they return empty text for these messages. You cannot rely on the Slack MCP to read the bot message body.

### Where the ticket ID comes from

When the Cursor Automation triggers on a Slack message, it injects the **full message payload** (including Block Kit blocks and attachments) into your prompt as trigger context. The ticket URL is embedded inside the blocks/attachments.

**Parse the trigger context** using this priority order:

1. **URL pattern** — scan the entire trigger context (blocks, attachments, fallback text, action URLs) for:
   ```
   https://<subdomain>.zendesk.com/agent/tickets/(\d+)
   ```
   This is the most reliable source. The URL is typically inside a Block Kit `section` block or an `attachment` with a `title_link` or `text` field.

2. **Hash pattern** — look for `#(\d+)` in any text field within the trigger context.

3. **Thread fallback** — if neither pattern matched in the trigger context, call `slack_read_thread` to check if a human reply in the thread mentions the ticket ID or URL. Bot messages will show empty, but human replies will contain readable text.
   ```
   tool: slack_read_thread (plugin-slack-slack)
   channel_id: <channel ID from trigger>
   message_ts: <message_ts from trigger>
   ```

If no ticket ID is found after all three attempts, reply to the thread:
> ⚠️ Could not extract a Zendesk ticket ID from this message. Please share the ticket number so I can investigate.

Then **stop**.

---

## Step 2 — Fetch the Zendesk ticket

Call both tools to get full context:

```
tool: zendesk_get_ticket (user-zendesk)
ticket_id: <extracted ID>
```

```
tool: zendesk_get_ticket_details (user-zendesk)
ticket_id: <extracted ID>
```

Extract from the response:
- **Subject**
- **Requester / Publisher name** (from custom fields or requester info)
- **Topic / category** (from custom fields or tags)
- **Environment** (production, sandbox — from custom fields or tags)
- **Key identifiers** from the ticket body and comments: Player ID, Order ID, Session ID, Publisher ID, error messages
- **Tags** on the ticket (useful for classification)

**Fast-exit cases — handle before investigation:**
- **Operational request** (e.g. "add user to dashboard", "update config", "grant access"): skip investigation and reply with the action needed.
- **Single-player transient report** (one player, no order ID, issue not reproducible): reply suggesting basic troubleshooting first (clear cache, different browser, reopen store).

---

## Step 3 — Classify the issue

Classify into one of:

`Dashboard` · `Checkout/Payments` · `Store/Offers` · `Award` · `SDK Events` · `Mobile` · `API/Reporting` · `Auth` · `Other`

Use the classification to narrow all subsequent searches.

---

## Step 4 — Check playbooks in Confluence

### 4a — Search for matching playbooks

```
tool: searchConfluenceUsingCql (user-atlassian-mcp)
cql: space.key = "Support" AND (text ~ "[keyword1]" OR text ~ "[keyword2]") AND type = page
```

Always wrap OR conditions in parentheses — otherwise `space.key` restriction only applies to the first term.

If a Publisher Name was extracted, also run:
```
cql: space.key = "Support" AND title = "[PublisherName]" AND type = page
```

### 4b — Read matched pages

For every Confluence page that looks relevant, call:
```
tool: getConfluencePage (user-atlassian-mcp)
contentFormat: "markdown"
```

Do **not** skip this step — always read the full body before writing the output.

---

## Step 5 — Check expected behavior via Appcharge docs

Fetch the relevant `docs.appcharge.com` page based on the classification to determine if the described behavior is by design.

### Docs URL mapping by classification

| Classification | Primary docs pages |
|---|---|
| **Checkout/Payments** | [Create Checkout Session API](https://docs.appcharge.com/api-reference/checkout/checkout-session/create-checkout-session) · [About the Checkout](https://docs.appcharge.com/guides/checkout/about-the-checkout) · [High-Level Flow](https://docs.appcharge.com/guides/checkout/checkout-high-level-flow) · [Set Up Your Checkout](https://docs.appcharge.com/guides/checkout/set-up-your-checkout) · [Design Your Checkout](https://docs.appcharge.com/guides/checkout/design-your-checkout) · [About Coupons](https://docs.appcharge.com/guides/checkout/coupons/about-coupons) · [Checkout Webhooks](https://docs.appcharge.com/api-reference/checkout/checkout-webhooks) |
| **Store/Offers** | [About the Web Store](https://docs.appcharge.com/guides/webstore/about-the-webstore) · [About the Template Studio](https://docs.appcharge.com/guides/webstore/design/template-studio/about-the-template-studio) · [About Offers](https://docs.appcharge.com/guides/webstore/offers/about-offers) · [About Personalization](https://docs.appcharge.com/guides/webstore/personalization/about-personalization) · [Configure Personalization](https://docs.appcharge.com/guides/webstore/personalization/configure-personalization) · [Personalize Web Store Callback](https://docs.appcharge.com/api-reference/webstore/personalization/personalize-webstore-callback) · [Web Store High-Level Flow](https://docs.appcharge.com/guides/webstore/webstore-high-level-flow) |
| **Award** | [About Awards](https://docs.appcharge.com/guides/checkout/awards/about-awards) · [Grant Award Callback](https://docs.appcharge.com/api-reference/checkout/awards/grant-award-callback) · [Set Up Award Notifications](https://docs.appcharge.com/guides/checkout/awards/set-up-award-notifications) |
| **SDK Events** | [About the Events Center](https://docs.appcharge.com/guides/events/about-the-events-center) · [Events v2 Introduction](https://docs.appcharge.com/api-reference/events/v2/introduction) · [Events v1 Introduction](https://docs.appcharge.com/api-reference/events/v1/introduction) |
| **Auth** | [About Player Authentication](https://docs.appcharge.com/guides/webstore/player-authentication/about-player-authentication) · [Set Up Player Authentication](https://docs.appcharge.com/guides/webstore/player-authentication/set-up-player-authentication) · [Authenticate Player Callback](https://docs.appcharge.com/api-reference/webstore/player-authentication/authenticate-player-callback) · [About SSO Login](https://docs.appcharge.com/guides/webstore/player-authentication/sso-login/about-sso-login) |
| **Dashboard** | [About the Publisher Dashboard](https://docs.appcharge.com/guides/publisher-dashboard/about-the-publisher-dashboard) · [View Analytics](https://docs.appcharge.com/guides/publisher-dashboard/view-analytics) · [View Orders](https://docs.appcharge.com/guides/publisher-dashboard/view-orders) · [Manage User Roles and Permissions](https://docs.appcharge.com/guides/publisher-dashboard/manage-user-roles-and-permissions) |
| **Mobile** | [About Payment Links](https://docs.appcharge.com/guides/payment-links/about-payment-links) · [Payment Link Flow](https://docs.appcharge.com/sdks/payment-links/payment-link-flow) · [Troubleshooting](https://docs.appcharge.com/sdks/payment-links/troubleshooting) · [Frontend SDK Introduction](https://docs.appcharge.com/sdks/checkout/frontend-sdk/introduction) |
| **API/Reporting** | [Analytics Reporting API](https://docs.appcharge.com/api-reference/checkout/finance-and-analytics/analytics-reporting-api) · [Financial Reporting API](https://docs.appcharge.com/api-reference/checkout/finance-and-analytics/financial-reporting-api) · [Get Transactions](https://docs.appcharge.com/api-reference/checkout/finance-and-analytics/get-transactions) · [View Financial Reports](https://docs.appcharge.com/guides/publisher-dashboard/view-financial-reports) |

### How to use the mapping

1. Based on the classification from Step 3, fetch the **primary docs page** (first link in the row).
2. If the specific symptom relates to a sub-topic (e.g. coupons, personalization, SSO), also fetch the more specific page.
3. Compare the documented behavior with what the publisher is reporting. If the behavior matches the docs, it’s expected.

### Fallback

If the mapped URL returns 404 or the classification is `Other`, fetch `https://docs.appcharge.com/llms.txt` to locate the correct page.

If no page can be fetched at all, note it explicitly in the output and advise manual verification at `https://docs.appcharge.com/`.

---

## Step 6 — Search for past resolutions and known R&D cases

Run both searches **in parallel**.

### 6a — Confluence: past resolution write-ups

Search the Support space for documented resolutions of similar issues:
```
tool: searchConfluenceUsingCql (user-atlassian-mcp)
cql: space.key = "Support" AND (text ~ "[symptom keyword]" OR text ~ "[error keyword]") AND type = page
```

If a page looks like a past incident resolution, call `getConfluencePage` with `contentFormat: "markdown"` to read the full body. Surface any "how it was resolved" detail.

### 6b — Jira: open bugs matching this symptom

Before recommending an R&D escalation, check for existing ACDEV tickets:
```
tool: searchJiraIssuesUsingJql (user-atlassian-mcp)
cloudId: 38515cfd-c8c7-4d33-80dc-c473e0ae2550
jql: project = ACDEV AND text ~ "[symptom keywords]" AND created >= -270d ORDER BY created DESC
```

- **Open result** → link it instead of recommending a new ticket; include status and fix ETA if available
- **Done/Cancelled result** → flag as possible regression; include the original resolution
- **Nothing found** → omit the "Known R&D Cases" section entirely

---

## Step 7 — Build the reply message

Format the output using this template (standard markdown):

```
## Ticket #[ID] — [Subject]

**Publisher:** [Name] · **Environment:** [Production / Sandbox] · **Opened:** [date]
> [One-sentence plain-English summary of what the publisher is reporting]

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

### What happened
[2–3 sentences in plain English. Explain what the publisher is reporting and what you found. No jargon.]

### What to do
1. [First concrete action — e.g. "Ask the publisher to check their award endpoint."]
2. [Second action if needed]

---

### Past Resolutions
_(Omit section entirely if nothing found)_
- [How a similar issue was resolved]

---

### Known R&D Cases
_(Omit section entirely if nothing found)_
- **[ACDEV-XXX](link)** `Open` — [plain-English description]

---

### Evidence
_(Omit if nothing meaningful to add — max 2–5 bullets)_
- [Key finding from Confluence, Jira, or docs]
```

If `Playbook = None found` AND both Past Resolutions and Known R&D Cases are empty, append:
> ⚠️ **No playbook and no known case exist for this issue type.** If this ticket gets resolved, consider documenting the solution.

---

## Step 8 — Reply in the Slack thread

```
tool: slack_send_message (plugin-slack-slack)
channel_id: <channel ID from the trigger>
message: <formatted output from Step 7>
thread_ts: <message_ts of the triggering message>
```

Do **not** set `reply_broadcast: true` — keep the reply in-thread only.

---

## Guardrails

- **READ-ONLY on Zendesk:** never modify, comment on, or close Zendesk tickets.
- **READ-ONLY on Confluence:** never create or edit Confluence pages.
- **Single reply:** do not post more than one Slack reply per invocation.
- **No web searches with internal data:** never search the web using publisher IDs, player IDs, order IDs, ticket content, or any data from MCP responses.
- **Evidence discipline:** never label something a "known issue" or "expected behavior" without citing a specific Jira ticket, Confluence page, or doc page as proof. If inferring, say "possible" or "likely" and flag as a hypothesis.
- **Plain language:** write for a technical support person, not a developer. Avoid log field names, service names, and code-level jargon in the output.
- **Block Kit awareness:** do NOT attempt to read Zendesk bot messages via the Slack MCP — they will return empty. Rely on the automation trigger context for the message payload.

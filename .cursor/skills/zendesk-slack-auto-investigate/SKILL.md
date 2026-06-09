---
name: zendesk-slack-auto-investigate
description: >-
  Automated support investigation triggered by a Zendesk bot message in Slack.
  Extracts the ticket ID from the Slack message, fetches ticket details via Zendesk MCP,
  runs the full playbook investigation (classify → playbooks → expected behavior → past resolutions → known R&D cases),
  and replies in the same Slack thread with a short triage summary for the internal support team.
  Use when a Zendesk bot posts a new ticket notification in a Slack channel and the automation fires.
---

# Zendesk–Slack Auto-Investigate

Automated skill triggered by a **Zendesk bot message in Slack**.

Every message in `#publishers-support-cases` from **Appcharge Support** (`U07CXNMD0Q7`) is a new Zendesk ticket notification.

---

## MCP Tools Required

| Server | Tools used |
|---|---|
| `plugin-slack-slack` | `slack_read_thread`, `slack_read_channel`, `slack_send_message` |
| `user-zendesk` | `zendesk_get_ticket`, `zendesk_get_ticket_details` |
| `user-atlassian-mcp` | `searchConfluenceUsingCql`, `getConfluencePage`, `searchJiraIssuesUsingJql`, `getJiraIssue` |
| `plugin-atlassian-atlassian` | `searchConfluenceUsingCql`, `searchJiraIssuesUsingJql` (alternative) |

---

## Step 1 — Extract the Zendesk ticket ID

The Appcharge Support bot uses **Slack Block Kit**. Plain text is often empty; the ticket URL is in blocks or attachments.

**Try every source below in order. Do not stop after one empty result.**

### 1a — Trigger context (always try first)

Scan **all** fields the automation injects: message text, blocks, attachments, `title`, `title_link`, `url`, `fallback`, action URLs, metadata, permalink.

Match any of these patterns (case-insensitive, anywhere in the payload):

```
zendesk\.com/agent/tickets/(\d+)
/tickets/(\d+)
#(\d+)
```

Subdomains include `appcharge.zendesk.com` and `appchargehelp.zendesk.com`.

### 1b — Slack read thread (always try second)

```
tool: slack_read_thread (plugin-slack-slack)
channel_id: <channel ID from trigger>
message_ts: <message_ts of triggering message>
response_format: detailed
```

Parse the **entire** response — text, blocks, attachments, links. Block Kit fields may appear here even when plain text is empty.

### 1c — Slack read channel (third — narrow window)

If 1a and 1b found nothing, read messages around the trigger timestamp:

```
tool: slack_read_channel (plugin-slack-slack)
channel_id: <channel ID from trigger>
oldest: <trigger message_ts minus 1 second>
latest: <trigger message_ts plus 1 second>
response_format: detailed
limit: 5
```

Find the triggering message in the results and parse it the same way as 1a.

### If still not found

Reply to the thread:
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
3. Compare the documented behavior with what the publisher is reporting. If the behavior matches the docs, it's expected.

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

Check for existing ACDEV tickets:
```
tool: searchJiraIssuesUsingJql (user-atlassian-mcp)
cloudId: 38515cfd-c8c7-4d33-80dc-c473e0ae2550
jql: project = ACDEV AND text ~ "[symptom keywords]" AND created >= -270d ORDER BY created DESC
```

Use findings in Step 7 for the **Known issue?** line and **Relevant links** section.

---

## Step 7 — Build the reply message

**Audience:** Internal support team triage — not publisher-facing. Keep it short. The team handles the ticket themselves; do not include action items or next steps.

Format the output using this template (standard markdown):

```
## Ticket #[ID] — [Subject]

**Publisher:** [Name] · **Environment:** [Production / Sandbox] · **Category:** [Category]

Expected behavior? [Yes/No] · Appcharge at fault? [Yes/No/Unknown] · Playbook: [Name](link) / None found

**Known issue?** [Yes — brief reason / No known issue found]

**Summary:**
[Line 1: what happened + likely cause]
[Line 2: key finding — expected behavior or why]
[Line 3 (optional): one caveat only — omit if none]

**Relevant links:**
- [Link label](url) — [one-line why it matters]
_(Omit section if nothing found)_
```

### Summary rules

- **Up to 3 lines.** Each line is one short sentence. No bullet points inside the summary.
- **Line 1:** Publisher, symptom, environment, and likely cause.
- **Line 2:** Key finding — why this is expected/unexpected or what docs/playbook say.
- **Line 3 (optional):** One caveat only if critical (e.g. missing order ID). Omit if not needed.
- **No process narration** — do not explain how you investigated or list multiple hypotheses.

**Bad (too long):**
> SpaceGo reports that a production purchase from Argentina fails with "Payment could not be processed..." They suspect ARS may be the cause. The error is a generic PSP decline. The Confluence playbook for test cards is Sandbox-only. No Order ID was included, so root cause cannot be confirmed.

**Good (ticket #37405):**
> SpaceGo prod ARS payment declined — generic PSP error.
> Likely Sandbox test card used in Production (ARS test cards are Sandbox-only).
> Unconfirmed — no order ID.

### Known issue definition

Determine **Known issue?** from any match during investigation. Keep to **one short phrase** (not a sentence):

| Match type | Reply format |
|---|---|
| Open ACDEV Jira ticket | `Yes — ACDEV-XXXX (Open)` |
| Done/Cancelled Jira (possible regression) | `Possible regression — ACDEV-XXXX` |
| Confluence playbook with known gotcha | `Yes — [Playbook Name](link)` |
| Past resolution in Confluence | `Seen before — [page title](link)` |
| Nothing found | `No` |

Never say "known issue" without citing the source. Details go in **Relevant links**, not here.

### Relevant links

Include up to 3 links — docs, playbooks, or Jira. Short labels only. Omit the section if nothing useful was found.

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

- **Internal triage only:** output is for the internal support team glancing at the Slack thread — not publisher-facing. No "What to do" or action items.
- **READ-ONLY on Zendesk:** never modify, comment on, or close Zendesk tickets.
- **READ-ONLY on Confluence:** never create or edit Confluence pages.
- **Single reply:** do not post more than one Slack reply per invocation.
- **No web searches with internal data:** never search the web using publisher IDs, player IDs, order IDs, ticket content, or any data from MCP responses.
- **Evidence discipline:** never label something a "known issue" or "expected behavior" without citing a specific Jira ticket, Confluence page, or doc page as proof. If inferring, say "possible" or "likely" and flag as a hypothesis.
- **Plain language:** write for a technical support person, not a developer. Avoid log field names, service names, and code-level jargon in the output.
- **Ticket ID extraction:** always try trigger context AND Slack MCP (`slack_read_thread` with `detailed`, then `slack_read_channel`). Block Kit content may only appear in one source.
- **Brevity:** entire reply should fit on one phone screen. Summary must not exceed 3 lines.

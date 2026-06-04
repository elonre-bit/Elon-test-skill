---
name: publishers-support-auto-investigate
description: >-
  Simple smoke test. When triggered by a Slack message in #publishers-support-cases,
  posts a confirmation reply in the same thread to verify the automation is working end-to-end.
---

# Publishers Support — Auto Investigate (Smoke Test)

## Step 1 — Post a confirmation reply to the Slack thread

Reply to the triggering message in the same thread:

```
tool: slack_send_message (plugin-slack-slack)
channel_id: <channel ID from the trigger>
thread_ts: <message_ts of the triggering message>
message: ✅ Automation is working! I received your message and replied successfully.
```

Do **not** set `reply_broadcast: true` — keep the reply in-thread only.

Then stop.

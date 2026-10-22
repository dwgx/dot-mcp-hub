---
name: hub-qq
description: Read, send, reply to, and recall QQ messages through a self-hosted MCP hub, and page through full group history.
---

# hub-qq

Use this skill when the user asks about QQ group activity, wants a message sent or recalled, or wants group history summarized.

## Before acting

1. Call `hub_call({ tool: "hub_status", args: {} })` and confirm the QQ adapter is `active` and connected.
2. Call `hub_call({ tool: "qq_read", args: { action: "get_group_info", params: { group_id: "<id>" } } })` to confirm the group is allowlisted.

## Reading history

`qq_get_recent_messages` returns newest-last pages:

```
hub_call({ tool: "qq_get_recent_messages",
           args: { conversation_id: "group:<id>", limit: 200 } })
```

Take `next_before` from the response and pass it back as `before_time` until `has_more` is false. Do not claim the full history was read until that loop ends.

## Sending

- `qq_send_message` — plain text to an allowlisted group or the owner's private chat.
- `qq_reply_to_message` — reply to a persisted message; in groups it mentions the author.
- Every send needs a unique `idempotency_key`; retrying with the same key must not duplicate.

## Recall

`qq_recall_message` only recalls messages this bot sent. Anything else returns `FORBIDDEN`. A recall that times out is `AMBIGUOUS` — do not retry with a new key.

## Reporting

Quote the actual tool output (`message_id`, `status`). If a call was not made, say `[UNVERIFIED]` instead of implying success.

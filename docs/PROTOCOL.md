# DWGX Hub MCP Protocol — 2026-10-22

Status: live on `dwgx-hub-r03d-api` (127.0.0.1:8766), public via nginx + FurCDN.

## Endpoints

| URL | Auth | Tools | Events |
|---|---|---|---|
| `https://dwgx.top/hub/mcp` | Bearer token (`/var/lib/dwgx-capability-hub-r03d/secrets/mcp-token`) | all | yes |
| `https://dwgx.top/hub/mcp-open` | none | 4 read-only (static) | yes |

`/.well-known/oauth-protected-resource|oauth-authorization-server|openid-configuration` return a no-auth metadata JSON (ChatGPT probes these before connecting).

## Protocol eras

- **Legacy (2025-06-18)**: `initialize` → `notifications/initialized` → `tools/list` → `tools/call`. Session id in `mcp-session-id`.
- **Modern (2026-07-28)**: no handshake, no session. Per request:
  - `server/discover` → `{resultType:"complete", supportedVersions:["2026-07-28"], serverInfo:{name,version}, capabilities:{tools:{},events:{}}}`
  - `tools/list` → **must** include `resultType:"complete"` (root cause of the ChatGPT plugin failure; without it the plugin reports "Unable to load tools")
  - `events/list` → `{resultType:"complete", events:[...], nextCursor}`
  - `params._meta` carries `io.modelcontextprotocol/protocolVersion` + `clientCapabilities`.

## Tools

Open (`/mcp-open`, static, read-only): `hub_status`, `hub_get_capabilities`, `qq_get_recent_messages`, `qq_get_connection_state`.

Token (`/mcp`): full set — QQ send/media/preview/forward, artifact upload, TTS, hub_read_file / hub_write_file (pending), hub_deploy, events_*.

## Events

Catalog: `qq.mention.created`, `qq.private_message.created`, `qq.connection.changed`, plus job events.

Subscribe:
```json
{"method":"events/subscribe","params":{
  "name":"qq.private_message.created",
  "arguments":{"conversation_id":"private:136666451"},
  "delivery":{"mode":"webhook","url":"https://<callback>","secret":"whsec_<base64 24-64 bytes>"},
  "ttlMs":86400000}}
```
Server flow: validate args → validate callback URL (HTTPS, public IP, no redirects) → POST `{"type":"verification","challenge":"..."}` signed with Standard Webhooks → require 2xx + echoed challenge → store subscription → return `{id, refreshBefore, cursor:null, truncated:false}`.

Delivery: QQ message persisted → `recordEvent` → `event_deliveries` row (pending) → 2s loop → signed POST:
headers `webhook-id` (= eventId), `webhook-timestamp`, `webhook-signature`, `X-MCP-Subscription-Id`; body `{eventId,name,timestamp,data}`.

## Verified 2026-10-22

- `server/discover` / `tools/list` / `events/list` via CDN: 200, correct `resultType`.
- ChatGPT plugin: connected (screenshot from Owner).
- Dot calls `/mcp` with token: `qq_get_recent_messages`, `qq_send_message` observed in journal.

## Known gaps

- Open endpoint exposes only 4 read-only tools; write tools stay token-only.
- `events/subscribe` end-to-end webhook delivery not yet exercised with a real ChatGPT callback (loop added, unit path untested).

## Security posture (2026-10-22)

- `/mcp-open` rejects JSON-RPC batch arrays (400) — batches bypass per-envelope allowlists.
- `/mcp-open` returns 403 for `events/subscribe` / `events/unsubscribe`; subscriptions require the token endpoint.
- `/mcp-open` body limit 1 MB; `/mcp` keeps the 71 MB upload limit.
- Auth middleware runs before `express.json`, so unauthenticated bodies are never buffered.
- Verified: array→400, non-allowlisted→403, tools/list→200, events/subscribe→403, token tools/list→200, token tools/call→200, no-token→401, health→200.
- Full record: `HUB-SECURITY-REVIEW-20261022.md`.

## Tool count

`/mcp` (token): 49 tools as of 2026-10-22, including the stable façade `hub_call` (forwards to any tool by name), `qq_recall_message`, `qq_get_outbound`, `dot_selftest`, `example_echo`.
`/mcp-open`: 4 static read-only tools.

## History pagination (2026-10-22)

`qq_get_recent_messages` accepts `limit` (max 200), `before_time`, `after_time` and returns
`next_before` + `has_more`. Backfill from NapCat (`get_group_msg_history`, count 5000) imported
2513 messages; the group now holds 2514 messages spanning 2026-08-24 → 10-22. A daily 04:15 cron
(`/usr/local/bin/dwgx-qq-history-import.mjs`) re-runs the backfill.

## Stable façade

`hub_call({ tool, args })` validates `args` against the target tool's schema and forwards to its
handler. It exists because ChatGPT freezes the plugin catalog at creation; new server-side tools
are reachable through it without rebuilding the plugin.

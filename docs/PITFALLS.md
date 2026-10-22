# Building a custom MCP server for ChatGPT dots: what the docs don't say

Field notes from wiring a ChatGPT dot (GPT-6 Astra, cloud computer) to a self-hosted MCP hub over a public HTTPS endpoint. Every item below cost hours; none is stated clearly in the public docs.

## 1. MCP 2.0 (2026-07-28) is stateless, and results need `resultType`

ChatGPT's MCP client sends `server/discover` → `tools/list` → `events/list` **without `initialize` and without a session id**. The official SDK's `StreamableHTTPServerTransport` expects a handshake and answers `tools/list` with HTTP 400 when no session exists.

Fix: serve each request with a throwaway server+transport, and stamp `resultType: "complete"` onto every result:

```js
if (!sid) {
  const srv = buildServer();
  const t = new StreamableHTTPServerTransport({ sessionIdGenerator: undefined, enableJsonResponse: false });
  await srv.connect(t);
  await t.handleRequest(req, res, body);
  await t.close();
  return;
}
```

Without `resultType`, the plugin form reports the useless message **"服务器未能完成 MCP 设置"** / "Unable to load tools" — with no hint about which field is missing. `server/discover`, `tools/list`, `tools/call`, `events/list` all need it.

## 2. `initialize` must echo the requested protocol version

If the client asks for `2026-07-28` and your server replies `2025-11-25` (the SDK's max), a strict client treats the server as too old and refuses tool calls. Mirror the requested version in the `initialize` result.

## 3. Auth: ChatGPT only offers OAuth or "no authentication"

There is no Bearer-header field. If you want a token, put it in the URL query and accept it server-side:

```
https://your.host/mcp?token=<secret>
```

Accept `Authorization: Bearer` **or** `?token=` on the server.

## 4. The dot's cloud computer has two different networks

From the dot's own terminal:

- **managed shell**: outbound HTTPS works through an OpenAI-managed `localhost` proxy (HTTP 200).
- **Cloud Desktop** (the yellow GUI desktop): a *different* Linux network namespace. Direct IPv4 TCP from its apps returns `ECONNREFUSED`; the shell's own raw TCP has returned `ENETUNREACH`.

Consequences: you cannot `apt install` (no sudo), native GUI apps have no network, and the shell's proxy is not reachable from the Desktop's namespace. User-space extraction (`dpkg-deb -x`) is the only way to run GUI apps.

## 5. Events need a public HTTPS callback the dot cannot host

MCP Events (`events/subscribe`) requires a `delivery.url` reachable over public HTTPS with Standard Webhooks signatures. The dot machine has no documented inbound endpoint, so it cannot receive its own webhooks. Polling (or a relay you control) is the practical default; do not promise real-time wake-up until you have a callback that actually receives a signed POST.

## 6. The dot can connect exactly one personal computer

Documented: one computer at a time, it must be online with the ChatGPT app open. That gives files/apps/tasks access — command execution is not documented.

## 7. The plugin's "Refresh" flow is mandatory after tool changes

Tool names, descriptions, schemas, and annotations are cached at install time. After changing the server, open the plugin's details page and refresh; otherwise the old tool surface stays.

## Minimal working checklist

- [ ] `server/discover` returns `{resultType:"complete", supportedVersions:["2026-07-28"], capabilities:{tools:{},events:{}}}`
- [ ] `initialize` echoes the requested version
- [ ] `tools/list` / `tools/call` / `events/list` all carry `resultType`
- [ ] `?token=` accepted in addition to the Authorization header
- [ ] no `$schema` assumptions — a static, small tool list is the safest first install
- [ ] events: real callback tested with a signed verification POST before claiming "subscribed"

Written 2026-10-22 from a working deployment; corrections welcome.

# dot-mcp-hub

A working, self-hosted MCP hub that a ChatGPT dot can drive: QQ message read/send/recall, media, event subscriptions, and server-side self-iteration.

This repo is the field manual. It exists because the public docs and the dot itself never state the boundaries that cost the most time.

## What is in here

| Path | What it covers |
|---|---|
| [`docs/PITFALLS.md`](docs/PITFALLS.md) | Seven failures that are not in the docs, with fixes |
| [`docs/PROTOCOL.md`](docs/PROTOCOL.md) | Endpoint contract: legacy 2025 handshake, modern 2026-07-28 stateless flow, events, security posture |
| [`docs/OBSIDIAN-MEMORY.md`](docs/OBSIDIAN-MEMORY.md) | Vault layout and frontmatter for using Obsidian as an agent memory store |
| [`plugin/`](plugin/) | A portable Agent Plugins package (manifest + skill) that wraps the hub |

## The three facts that matter most

1. **ChatGPT freezes the tool catalog at plugin creation.** "Refresh tools" does not re-fetch. Ship one stable façade tool (`hub_call`) and add capabilities server-side.
2. **MCP 2026-07-28 is stateless and every result needs `resultType: "complete"`.** Without it the plugin form reports a generic failure with no hint.
3. **The dot's cloud computer has two networks.** The managed shell reaches HTTPS; the Cloud Desktop namespace does not (native IPv4 `ECONNREFUSED`), and there is no sudo.

## Shape of the server

- One stable façade: `hub_call({ tool, args })` — validates args against the target schema, then forwards.
- Self-iteration: `hub_write_file` stages a patch, `hub_deploy` runs tests → backup → restart → health gate → rollback.
- Events: webhook delivery with Standard Webhooks signatures; the platform receives the callback, so the dot needs no inbound endpoint.
- Read-only actions are separated from writes, and the unauthenticated endpoint exposes only a fixed read-only subset.

## Quick start

```bash
# 1. install deps
npm install

# 2. run
HUB_DATA=/var/lib/hub HUB_PORT=8766 node app/server.mjs

# 3. point ChatGPT at it
#    ChatGPT Plugins -> Add custom MCP server -> https://your.host/mcp?token=<secret>
#    Authentication: No authentication (the token rides in the query string)
```

## License

MIT. The pitfalls doc is meant to be copied into issues and support tickets.

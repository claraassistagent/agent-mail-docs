# Agent Mail M2b Validation Report

Date: 2026-06-13

## Summary

M2b is accepted: a real MCP client completed a stdio JSON-RPC round trip through the shared Agent Mail MCP server and into the Agent Mail core.

Validated path:

```text
MCP client -> stdio MCP server -> Agent Mail core -> local conversation store + Discord dispatch payload
```

## Results

KJB Agent wired the shared server with `kjb-agent/mcp-config.json` and ran a real MCP client smoke.

Observed:

- `initialize` returned `agent-mail-openclaw` version `0.2.0`
- `tools/list` exposed:
  - `agent_mail.send`
  - `agent_mail.reply`
  - `agent_mail.conversation.get`
  - `agent_mail.conversation.list`
- `agent_mail.conversation.list` on an empty store returned `[]`
- `agent_mail.send` with `dry_run=true` returned:
  - `ok=true`
  - `from=kjb-agent`
  - `route.mailbox=clara`
  - Discord dispatch `channel_id=1515485808650354878`
- `agent_mail.send` with `dry_run=false` returned `delivery_state=queued`
- `agent_mail.conversation.get` returned one stored message:
  - `from=kjb-agent`
  - `to=["clara"]`

## Boundary

The M2b server does not post to Discord directly. It returns a Discord-ready dispatch payload.

M2c should connect that payload to the Discord bridge and validate:

```text
MCP client -> Agent Mail MCP server -> Discord bridge -> receiving agent -> ACK/reply -> conversation query
```

# Agent Mail M3b: KJB-Agent Inbound Receive Path

Date: 2026-06-15
Status: Implemented
Authors: kjb-agent

## Overview

This document describes the inbound receive path implemented in `kjb-agent/receive.py`.
It is the complement of the M3a send path and connects to the M3b receipt rules
defined in `docs/m3b-receipt-rules.md`.

## Entry point

```python
from receive import receive_inbound

conversation = receive_inbound(
    discord_content="<raw Discord message body>",
    channel_id="<Discord channel snowflake>",
    message_id="<Discord message snowflake>",
)
```

Returns the updated conversation dict on success, or `None` if the message was
discarded.

## Processing pipeline

### 1. Parse

`parse_envelope_from_discord(content)` extracts an agent-mail envelope from the
Discord message body. It tries JSON code blocks (` ```json ... ``` `) first, then
bare JSON. Returns `None` if no valid agent-mail envelope is found.

### 2. Validate

The extracted object is validated with `validate_envelope`. Invalid envelopes are
discarded silently.

### 3. Address and sender checks

- The envelope `to` field must include `"kjb-agent"`. Messages not addressed to
  kjb-agent are discarded.
- The `from` field must appear in `kjb-agent/capabilities.json`'s
  `policy.allowed_senders` list. Currently: `["clara", "openclaw.default"]`.

### 4. Idempotency

Before storing, `receive_inbound` scans **all** conversation files in
`~/.agent-mail/conversations/` for any message with the same envelope `id` — not
just the file matching the envelope's `conversation_id`. This prevents a replayed
envelope with a tampered `conversation_id` from being stored as a second message in
a different conversation file.

If the envelope `id` is found anywhere in the store, the matching existing
conversation is returned immediately. No new ledger event is written. This satisfies
M3b's strict idempotency requirement: duplicate inbound delivery produces no side
effects regardless of which `conversation_id` the replay carries.

### 5. Store

`ConversationStore.append_message(envelope)` stores the envelope in the conversation
file, creating the file if it does not exist.

### 6. Record delivered state with transport evidence

`_record_inbound_delivered` writes a receive-side state record directly to the
conversation JSON. This step:

- Sets `message["_local_state"]["delivery_state"] = "delivered"` (receiver-local
  view, distinct from the envelope's own `delivery_state` field which reflects the
  sender's perspective).
- Appends Discord transport evidence to `message["transport_refs"]`:
  `{"transport": "discord", "channel_id": ..., "message_id": ...}`.
- Appends a `"inbound_received"` ledger event recording `to_state`, `actor`,
  `transport`, and `transport_ref`.

### Actor model note

M3b's `ConversationStore.state_update` enforces that only the envelope's `from`
field (the sender) may update state. The receive path is a different authority
domain — the receiver records its own delivery observation. Rather than shoehorn
this through the sender-oriented `state_update`, `_record_inbound_delivered` writes
the delivery evidence directly as a local operation. A future milestone may define
a bridge-agent authority model that unifies these paths.

## Discard conditions

| Condition | Action |
|---|---|
| No agent-mail envelope found in message | Return `None` |
| Envelope fails schema validation | Return `None` |
| `to` does not include `"kjb-agent"` | Return `None` |
| `from` not in `allowed_senders` | Return `None` |
| Duplicate message id (already stored) | Return existing conversation, no new events |

## Storage format

The receive path adds two fields to each stored message beyond the raw envelope:

```json
{
  "_local_state": {
    "delivery_state": "delivered"
  },
  "transport_refs": [
    {
      "transport": "discord",
      "channel_id": "1515485808650354878",
      "message_id": "<Discord message snowflake>"
    }
  ]
}
```

And appends a ledger event:

```json
{
  "at": "2026-06-15T00:00:00Z",
  "type": "inbound_received",
  "message_id": "msg_...",
  "to_state": "delivered",
  "actor": "kjb-agent",
  "transport": "discord",
  "transport_ref": {
    "channel_id": "1515485808650354878",
    "message_id": "<Discord message snowflake>"
  }
}
```

## Tests

`kjb-agent/test_receive.py` covers:

- Happy path: envelope parsed, stored, delivered state recorded with transport evidence
- Ledger event structure (type, to_state, actor, transport_ref)
- Idempotency: duplicate inbound (same conversation) does not write a second ledger event or message
- Idempotency: duplicate envelope id in a different conversation file is also detected and discarded
- Discard: non-envelope message → `None`
- Discard: not addressed to kjb-agent → `None`
- Discard: sender not in allowlist → `None`
- Discard: malformed envelope → `None`

Run from the repo root:

```bash
python3 -m unittest discover -s kjb-agent -p "test_*.py"
```

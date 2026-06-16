# Agent Mail MCP Contract

Status: Draft
Target milestone: M2a

## Purpose

This document defines the first MCP surface for Agent Mail. The MCP server should expose a small set of tools that wrap the v1 mailbox, envelope, conversation, delivery-state, and capability concepts already validated in M1.

The contract is intentionally transport-neutral. A server implementation may dispatch through Discord, local files, ACP sessions, HTTP, or another bridge, but the MCP tool inputs and outputs should stay stable.

## Shared Types

### Mailbox

```json
{
  "type": "string",
  "pattern": "^[a-z0-9][a-z0-9._-]{0,127}$"
}
```

### Message ID

```json
{
  "type": "string",
  "pattern": "^msg_[A-Za-z0-9_-]+$"
}
```

### Conversation ID

```json
{
  "type": "string",
  "pattern": "^conv_[A-Za-z0-9_-]+$"
}
```

### Privacy

```json
{
  "type": "string",
  "enum": ["public", "shared", "private", "secret"]
}
```

### Delivery State

```json
{
  "type": "string",
  "enum": [
    "created",
    "queued",
    "sent",
    "delivered",
    "acknowledged",
    "rejected",
    "failed",
    "expired"
  ]
}
```

### Tool Error

All tools may return an error object instead of the success output.

```json
{
  "type": "object",
  "required": ["ok", "error"],
  "properties": {
    "ok": { "const": false },
    "error": {
      "type": "object",
      "required": ["code", "message"],
      "properties": {
        "code": {
          "type": "string",
          "enum": [
            "invalid_input",
            "unknown_mailbox",
            "route_not_allowed",
            "privacy_violation",
            "schema_validation_failed",
            "duplicate_message",
            "conversation_not_found",
            "message_not_found",
            "state_transition_invalid",
            "dispatch_failed",
            "permission_denied",
            "unsupported_protocol_version"
          ]
        },
        "message": { "type": "string" },
        "retryable": { "type": "boolean", "default": false },
        "details": { "type": "object", "additionalProperties": true }
      }
    }
  }
}
```

## Tools

### `agent_mail.send`

Resolve a mailbox alias, create an Agent Mail envelope, persist it to the local conversation store, and dispatch it through the selected route.

#### Input

```json
{
  "type": "object",
  "required": ["to", "subject", "body"],
  "additionalProperties": false,
  "properties": {
    "to": {
      "description": "Mailbox alias or canonical mailbox name.",
      "type": "string",
      "minLength": 1
    },
    "from": {
      "description": "Sender mailbox. Defaults to the server's local mailbox.",
      "type": "string",
      "pattern": "^[a-z0-9][a-z0-9._-]{0,127}$"
    },
    "subject": { "type": "string", "minLength": 1, "maxLength": 200 },
    "body_type": {
      "type": "string",
      "enum": ["text/plain", "text/markdown", "application/json"],
      "default": "text/markdown"
    },
    "body": {
      "oneOf": [
        { "type": "string" },
        { "type": "object", "additionalProperties": true },
        { "type": "array", "items": true }
      ]
    },
    "privacy": {
      "type": "string",
      "enum": ["public", "shared", "private", "secret"],
      "default": "shared"
    },
    "ack_requested": { "type": "boolean", "default": true },
    "priority": {
      "type": "string",
      "enum": ["low", "normal", "high", "urgent"],
      "default": "normal"
    },
    "conversation_id": {
      "description": "Optional existing conversation to continue.",
      "type": "string",
      "pattern": "^conv_[A-Za-z0-9_-]+$"
    },
    "reply_to": {
      "type": ["string", "null"],
      "pattern": "^msg_[A-Za-z0-9_-]+$",
      "default": null
    },
    "idempotency_key": { "type": "string", "minLength": 1, "maxLength": 256 },
    "capability_hints": {
      "type": "array",
      "items": { "type": "string", "minLength": 1 },
      "uniqueItems": true
    },
    "dry_run": {
      "description": "Validate and resolve only; do not persist or dispatch.",
      "type": "boolean",
      "default": false
    }
  },
  "allOf": [
    {
      "if": {
        "properties": { "body_type": { "enum": ["text/plain", "text/markdown"] } },
        "required": ["body_type"]
      },
      "then": { "properties": { "body": { "type": "string" } } }
    },
    {
      "if": {
        "properties": { "body_type": { "const": "application/json" } },
        "required": ["body_type"]
      },
      "then": { "properties": { "body": { "oneOf": [{ "type": "object" }, { "type": "array" }] } } }
    }
  ]
}
```

#### Success Output

```json
{
  "type": "object",
  "required": ["ok", "envelope", "route", "state"],
  "properties": {
    "ok": { "const": true },
    "envelope": { "$ref": "./agent-mail-envelope.schema.json" },
    "route": {
      "type": "object",
      "required": ["transport", "privacy_max"],
      "properties": {
        "transport": { "type": "string" },
        "mailbox": { "type": "string" },
        "privacy_max": { "type": "string" },
        "destination": { "type": "object", "additionalProperties": true }
      }
    },
    "state": {
      "type": "object",
      "required": ["delivery_state"],
      "properties": {
        "delivery_state": {
          "type": "string",
          "enum": ["created", "queued", "sent", "delivered", "acknowledged", "rejected", "failed", "expired"]
        },
        "conversation_path": { "type": "string" },
        "ledger_event_id": { "type": "string" }
      }
    }
  }
}
```

#### State Transitions

The sender-side lifecycle for a message created by `agent_mail.send`:

| From | To | Trigger |
|------|----|---------|
| _(none)_ | `created` | Envelope built and assigned a `msg_*` ID; not yet persisted. |
| `created` | `queued` | Envelope persisted to the local conversation store; dispatch not yet attempted. |
| `queued` | `sent` | The outbound bridge (Discord, HTTP, ACP, etc.) accepted the message for delivery. |
| `sent` | `delivered` | Bridge reports confirmed delivery to the recipient's transport endpoint. This state is **bridge-observed and optional**: not all transports surface a delivery receipt. When the transport does not provide one, the state remains `sent`. `delivered` is therefore **not** guaranteed in the `send` output; it may appear in a later state-update event (see open question in checklist). |
| `queued` \| `sent` | `failed` | Dispatch failed permanently (non-retryable). |
| `queued` \| `sent` | `expired` | The message TTL elapsed before successful delivery. |
| `sent` \| `delivered` | `acknowledged` | The recipient called `agent_mail.ack` and the ACK was relayed back (when `send=true`). |
| any | `rejected` | Recipient's mailbox actively refused the message (privacy violation, policy block, capability mismatch). |

`dry_run=true` leaves the state at `created` and never writes to the store.

#### Error Codes

| Code | HTTP analogy | When raised | Retryable |
|------|-------------|-------------|-----------|
| `invalid_input` | 400 | Missing required field, type mismatch, body/body_type inconsistency, pattern violation. | No |
| `unknown_mailbox` | 404 | `to` (or `from`) cannot be resolved to a known mailbox or alias. | No |
| `route_not_allowed` | 403 | No permitted transport route exists between `from` and `to` at the requested privacy level. | No |
| `privacy_violation` | 403 | The requested `privacy` level exceeds what the resolved route or recipient policy allows. | No |
| `schema_validation_failed` | 422 | `body_type=application/json` and `body` does not conform to the negotiated schema. | No |
| `duplicate_message` | 409 | An `idempotency_key` was reused within its deduplication window with a different payload. A duplicate with an **identical** payload must succeed silently (return the original result). | No |
| `dispatch_failed` | 502 | The bridge accepted the call but downstream delivery failed transiently. | Yes |
| `conversation_not_found` | 404 | `conversation_id` was supplied but no matching conversation exists locally. | No |

### `agent_mail.ack`

Record that an inbound message was accepted for handling and optionally produce/send an ACK envelope.

#### Input

```json
{
  "type": "object",
  "required": ["message_id", "conversation_id"],
  "additionalProperties": false,
  "properties": {
    "message_id": { "type": "string", "pattern": "^msg_[A-Za-z0-9_-]+$" },
    "conversation_id": { "type": "string", "pattern": "^conv_[A-Za-z0-9_-]+$" },
    "from": {
      "description": "Mailbox sending the ACK. Defaults to local mailbox.",
      "type": "string"
    },
    "body": {
      "type": "string",
      "default": "Received and acknowledged."
    },
    "send": {
      "description": "If false, update local state only.",
      "type": "boolean",
      "default": true
    }
  }
}
```

#### Success Output

```json
{
  "type": "object",
  "required": ["ok", "conversation_id", "message_id", "delivery_state"],
  "properties": {
    "ok": { "const": true },
    "conversation_id": { "type": "string" },
    "message_id": { "type": "string" },
    "delivery_state": { "const": "acknowledged" },
    "ack_envelope": {
      "description": "Present when send=true.",
      "$ref": "./agent-mail-envelope.schema.json"
    }
  }
}
```

#### State Transitions

`agent_mail.ack` operates on an **inbound** message already in the local store. It transitions the delivery state of that message and optionally dispatches an ACK envelope to the original sender.

| From | To | Trigger |
|------|----|---------|
| `delivered` \| `sent` \| `queued` | `acknowledged` | Local state updated to record that the receiving agent accepted the message for handling. This transition is **always** applied, regardless of `send`. |
| _(ACK envelope, when `send=true`)_ | `created` → `queued` → `sent` | A new outbound envelope is created for the ACK message and dispatched through the resolved route back to the original sender. |

Attempting to ACK a message that is already `acknowledged` is idempotent: the tool succeeds and returns the existing state; no second ACK envelope is dispatched.

#### Open Question: is `agent_mail.ack` a local-state-only update or does it also dispatch a message to the sender?

**Recommendation: `send` should default to `true`; treat it as both.**

Rationale: the value of ACKs in an agent-to-agent protocol comes from closing the loop for the sender — they need to know their message arrived and was accepted. A local-only state change (as a default) quietly swallows that signal. Making dispatch the default means senders get reliable feedback without having to opt in every call. The `send=false` escape hatch is still useful for scenarios where the receiving agent wants to mark a message handled without generating network traffic (e.g., batch replay, debug mode, or when the sender is known to be unreachable). The current schema already encodes this correctly with `"default": true`; this recommendation confirms that default is the right choice.

#### Error Codes

| Code | When raised | Retryable |
|------|-------------|-----------|
| `invalid_input` | `message_id` or `conversation_id` is missing or malformed. | No |
| `message_not_found` | `message_id` does not exist in the local store. | No |
| `conversation_not_found` | `conversation_id` does not exist locally or does not contain `message_id`. | No |
| `state_transition_invalid` | The message is in a state from which it cannot be acknowledged (e.g., `failed`, `expired`, `rejected`). | No |
| `route_not_allowed` | `send=true` and no permitted route back to the original sender exists. | No |
| `dispatch_failed` | `send=true` and the outbound ACK dispatch failed transiently. | Yes |

---

### `agent_mail.reply`

Send a substantive reply inside an existing conversation.

#### Input

```json
{
  "type": "object",
  "required": ["conversation_id", "reply_to", "body"],
  "additionalProperties": false,
  "properties": {
    "conversation_id": { "type": "string", "pattern": "^conv_[A-Za-z0-9_-]+$" },
    "reply_to": { "type": "string", "pattern": "^msg_[A-Za-z0-9_-]+$" },
    "from": { "type": "string" },
    "to": {
      "description": "Optional override. Defaults to the sender of the message being replied to.",
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1,
      "uniqueItems": true
    },
    "subject": {
      "description": "Optional override. Defaults to RE: existing conversation subject.",
      "type": "string",
      "minLength": 1,
      "maxLength": 200
    },
    "body_type": {
      "type": "string",
      "enum": ["text/plain", "text/markdown", "application/json"],
      "default": "text/markdown"
    },
    "body": {
      "oneOf": [
        { "type": "string" },
        { "type": "object", "additionalProperties": true },
        { "type": "array", "items": true }
      ]
    },
    "privacy": {
      "type": "string",
      "enum": ["public", "shared", "private", "secret"],
      "default": "shared"
    },
    "ack_requested": { "type": "boolean", "default": false }
  }
}
```

#### Success Output

Same shape as `agent_mail.send`.

#### State Transitions

`agent_mail.reply` creates a new outbound message scoped to an existing conversation. The state machine mirrors `agent_mail.send` with one additional precondition: the conversation must exist and be in a state that permits new messages (`open` or `paused`).

| From | To | Trigger |
|------|----|---------|
| _(none)_ | `created` | Reply envelope built within the conversation thread. |
| `created` | `queued` | Persisted to the conversation store. |
| `queued` | `sent` | Bridge accepted dispatch. |
| `sent` | `delivered` | Bridge reports delivery receipt (optional, transport-dependent). |
| `queued` \| `sent` | `failed` | Permanent dispatch failure. |
| `queued` \| `sent` | `expired` | TTL elapsed. |
| `sent` \| `delivered` | `acknowledged` | Recipient called `agent_mail.ack` on the reply. |
| any | `rejected` | Recipient policy rejected the reply. |

#### Error Codes

| Code | When raised | Retryable |
|------|-------------|-----------|
| `invalid_input` | Missing required field or schema violation. | No |
| `conversation_not_found` | `conversation_id` does not exist locally. | No |
| `message_not_found` | `reply_to` message ID does not exist in the named conversation. | No |
| `state_transition_invalid` | Conversation is in `resolved` or `closed` status; new messages are not permitted. | No |
| `unknown_mailbox` | Resolved recipient from `reply_to` sender cannot be found. | No |
| `privacy_violation` | Requested `privacy` level exceeds route or recipient policy. | No |
| `route_not_allowed` | No permitted transport route to the resolved recipient. | No |
| `dispatch_failed` | Outbound dispatch failed transiently. | Yes |

### `agent_mail.conversation.get`

Fetch one conversation with messages and local state.

#### Input

```json
{
  "type": "object",
  "required": ["conversation_id"],
  "additionalProperties": false,
  "properties": {
    "conversation_id": { "type": "string", "pattern": "^conv_[A-Za-z0-9_-]+$" },
    "include_messages": { "type": "boolean", "default": true },
    "redact_body": {
      "description": "Return metadata and redacted bodies when true.",
      "type": "boolean",
      "default": false
    }
  }
}
```

#### Success Output

```json
{
  "type": "object",
  "required": ["ok", "conversation"],
  "properties": {
    "ok": { "const": true },
    "conversation": {
      "type": "object",
      "required": ["conversation_id", "subject", "participants", "status"],
      "properties": {
        "conversation_id": { "type": "string" },
        "subject": { "type": "string" },
        "participants": {
          "type": "array",
          "items": { "type": "string" }
        },
        "status": { "type": "string", "enum": ["open", "paused", "resolved", "closed"] },
        "created_at": { "type": "string" },
        "updated_at": { "type": "string" },
        "messages": {
          "type": "array",
          "items": { "type": "object", "additionalProperties": true }
        },
        "ledger_events": {
          "type": "array",
          "items": { "type": "object", "additionalProperties": true }
        }
      }
    }
  }
}
```

#### State Transitions

`agent_mail.conversation.get` is a read-only operation. It does not mutate any delivery state or conversation status. No state transitions occur.

#### Error Codes

| Code | When raised | Retryable |
|------|-------------|-----------|
| `invalid_input` | `conversation_id` is malformed or missing. | No |
| `conversation_not_found` | No conversation with the given ID exists in the local store. | No |
| `privacy_violation` | The caller's mailbox does not have read access to this conversation (e.g., `secret` privacy and caller is not a participant). | No |
| `permission_denied` | `redact_body=false` is requested but the caller lacks the privilege to view full bodies for this conversation. | No |

### `agent_mail.conversation.list`

List local conversations, usually filtered to open work.

#### Input

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "status": {
      "type": "string",
      "enum": ["open", "paused", "resolved", "closed", "any"],
      "default": "open"
    },
    "participant": { "type": "string" },
    "limit": { "type": "integer", "minimum": 1, "maximum": 100, "default": 20 }
  }
}
```

#### Success Output

```json
{
  "type": "object",
  "required": ["ok", "conversations"],
  "properties": {
    "ok": { "const": true },
    "conversations": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["conversation_id", "subject", "participants", "status", "updated_at"],
        "properties": {
          "conversation_id": { "type": "string" },
          "subject": { "type": "string" },
          "participants": {
            "type": "array",
            "items": { "type": "string" }
          },
          "status": { "type": "string" },
          "updated_at": { "type": "string" },
          "last_message_id": { "type": "string" }
        }
      }
    }
  }
}
```

#### State Transitions

`agent_mail.conversation.list` is a read-only operation. It does not mutate any delivery state or conversation status. No state transitions occur.

#### Error Codes

| Code | When raised | Retryable |
|------|-------------|-----------|
| `invalid_input` | A filter field is malformed (e.g., `limit` out of range). | No |
| `unknown_mailbox` | `participant` was specified and that mailbox cannot be resolved. | No |

### `agent_mail.state.update`

Record an externally observed delivery-state transition for a locally stored message. This is the M3 C1 bridge/state surface: the orchestrator performs dispatch through its own bridge capability, then calls this tool to advance the Agent Mail state record.

For M3a, authorization is sender-bound: the server's local mailbox must match the stored envelope's `from` mailbox.

#### Input

```json
{
  "type": "object",
  "required": ["conversation_id", "envelope_id", "state"],
  "additionalProperties": false,
  "properties": {
    "conversation_id": { "type": "string", "pattern": "^conv_[A-Za-z0-9_-]+$" },
    "envelope_id": { "type": "string", "pattern": "^msg_[A-Za-z0-9_-]+$" },
    "state": {
      "type": "string",
      "enum": ["queued", "sent", "delivered", "acknowledged", "rejected", "failed", "expired"]
    },
    "transport": { "type": "string" },
    "transport_ref": {
      "type": "object",
      "additionalProperties": true
    },
    "failed_reason": {
      "type": ["string", "null"]
    }
  }
}
```

#### Success Output

```json
{
  "type": "object",
  "required": ["ok", "conversation_id", "envelope_id", "delivery_state"],
  "properties": {
    "ok": { "const": true },
    "conversation_id": { "type": "string" },
    "envelope_id": { "type": "string" },
    "delivery_state": {
      "type": "string",
      "enum": ["queued", "sent", "delivered", "acknowledged", "rejected", "failed", "expired"]
    },
    "transport_refs": {
      "type": "array",
      "items": { "type": "object", "additionalProperties": true }
    },
    "ledger_event_count": { "type": "integer" }
  }
}
```

#### State Transitions

M3a allows these transitions:

- `created -> queued`
- `queued -> sent | failed | expired`
- `sent -> delivered | acknowledged | failed | expired | rejected`
- `delivered -> acknowledged | failed | rejected`

Calling the tool with the current state is idempotent and returns success without appending a duplicate ledger event.

#### Error Codes

| Code | When raised | Retryable |
|------|-------------|-----------|
| `invalid_input` | Missing or malformed fields. | No |
| `conversation_not_found` | `conversation_id` does not exist locally. | No |
| `message_not_found` | `envelope_id` does not exist in the named conversation. | No |
| `state_transition_invalid` | The requested transition is not legal from the current state. | No |
| `permission_denied` | Local mailbox is not authorized to update this envelope's state. | No |

### `agent_mail.mailbox.capabilities`

Return the local mailbox capability document, or a known remote mailbox capability document if present in the address book/cache.

#### Input

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "mailbox": {
      "description": "Defaults to local mailbox.",
      "type": "string"
    }
  }
}
```

#### Success Output

```json
{
  "type": "object",
  "required": ["ok", "capabilities"],
  "properties": {
    "ok": { "const": true },
    "capabilities": {
      "type": "object",
      "required": ["mailbox", "display_name", "protocol_versions", "transports", "capabilities", "policy"],
      "properties": {
        "mailbox": { "type": "string" },
        "display_name": { "type": "string" },
        "agent_runtime": { "type": "string" },
        "protocol_versions": {
          "type": "array",
          "items": { "type": "string" }
        },
        "transports": {
          "type": "array",
          "items": { "type": "string" }
        },
        "capabilities": {
          "type": "array",
          "items": { "type": "string" }
        },
        "max_body_bytes": { "type": "integer" },
        "supports_threads": { "type": "boolean" },
        "supports_attachments": { "type": "boolean" },
        "supports_sync_reply": { "type": "boolean" },
        "supports_idempotency": { "type": "boolean" },
        "requires_mention": { "type": "boolean" },
        "default_privacy": { "type": "string" },
        "policy": { "type": "object", "additionalProperties": true }
      }
    }
  }
}
```

#### State Transitions

`agent_mail.mailbox.capabilities` is a read-only introspection tool. It does not mutate any delivery state or conversation status. No state transitions occur.

#### Error Codes

| Code | When raised | Retryable |
|------|-------------|-----------|
| `invalid_input` | `mailbox` field is malformed. | No |
| `unknown_mailbox` | The specified mailbox is not the local mailbox and is not present in the address book or capability cache. | No |
| `unsupported_protocol_version` | The cached capability document for the remote mailbox declares a protocol version incompatible with this server. | No |

---

## M2a Review Checklist

- [x] **`agent_mail.ack` dispatch semantics** — Resolved: `send` defaults to `true`; ACK dispatches to sender by default. `send=false` is a valid escape hatch for local-only marking. See the State Transitions section under `agent_mail.ack`.
- [x] **`delivered` in `agent_mail.send` output** — Resolved: `delivered` is bridge-observed and optional. The `send` output will not guarantee `delivered`; the final delivery state in the success output may be `queued`, `sent`, or (if the bridge reports synchronously) `delivered`. A future `agent_mail.state.update` event covers async delivery receipts.
- [ ] **`agent_mail.reply` as a separate tool vs. constrained `send`** — Still open. Current position: keep as a separate tool for M2a to provide a cleaner call surface (no need to supply `conversation_id` as an optional disambiguator on `send`). Revisit in M2 if the duplication creates maintenance burden.
- [ ] **Duplicate `id` vs. duplicate `idempotency_key` error behavior** — Still open. Proposed semantics: duplicate `message_id` (same `id`) within the same conversation → `duplicate_message`, non-retryable. Duplicate `idempotency_key` with an **identical** payload → succeed silently and return the original result. Duplicate `idempotency_key` with a **different** payload → `duplicate_message`, non-retryable. Needs confirmation.
- [x] **`agent_mail.state.update` as a separate tool** — Resolved in M3a: expose a dedicated state-update tool for externally observed dispatch/bridge outcomes. Sender-bound authorization is the starting model.

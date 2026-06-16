# Agent Mail M3b: Receipt Authority and Reconciliation Rules

Date: 2026-06-14
Status: Started
Authors: Clara
Target milestone: M3b

## ELI5 framing

M3a proved that Agent Mail can act like a mailbox with a receipt book:

1. An agent writes a standard letter.
2. The Agent Mail server stores it in a durable conversation.
3. An external delivery truck, such as Discord, carries it.
4. The sender records the receipt with `agent_mail.state.update`.
5. The receiver ACKs or replies.

M3b is about making the receipt book trustworthy. The protocol should make it harder for an agent to mark a message as `sent`, `delivered`, `acknowledged`, or `failed` without enough evidence for a human or another agent to audit later.

## Current trust boundary

The current implementation authenticates `agent_mail.state.update` through the server's configured local mailbox:

- The MCP server has one `local_mailbox`.
- The state-update actor is that configured mailbox.
- The actor may update state only for messages whose envelope `from` matches that mailbox.

This is intentionally narrow. It prevents a local Clara server from mutating KJB-authored messages, and vice versa. It is not yet a full protocol-level identity system. M3b should keep this caveat visible until a later milestone defines stronger caller identity.

## M3b receipt rules implemented in this slice

### 1. Evidence-required states

The following states require a `transport` and non-empty `transport_ref`:

- `sent`
- `delivered`
- `acknowledged`

Known transport refs must include stable receipt keys:

- `discord`: `channel_id`, `message_id`
- `acp`: `session_id`
- `http`: `url`, `status_code`
- `file`: `path`

Unknown transports are allowed for portability, but still require a non-empty `transport_ref`.

### 2. Failed states require a reason

`state=failed` requires `failed_reason`.

The reason should be machine-readable, for example:

- `discord_rate_limited`
- `route_unreachable`
- `transport_auth_failed`
- `recipient_not_found`
- `ttl_expired`

This slice records `failed_reason` and increments `retry_count` once when a message successfully transitions into `failed`. Rejected state-update calls do not increment it, and a duplicate `failed` update with the same reason remains idempotent.

### 3. Idempotency is strict, not contradictory

Repeating the same state update with the same state and same evidence is idempotent and does not write a duplicate ledger event.

Repeating the same state update with different evidence is rejected as `state_transition_invalid`. This avoids silently accepting contradictory receipts, such as two different Discord message IDs for the same `sent` transition.

## What remains for later M3b work

- Decide whether designated bridge agents can update sender-authored messages, and how that authority is represented.
- Define retry semantics beyond recording `retry_count`; the current state machine still treats `failed` as terminal.
- Decide whether `delivered` should be transport-observed only or may be inferred by a bridge.
- Define standard `failed_reason` values.
- Add cross-runtime compatibility fixtures for Discord, ACP, HTTP, and file-backed transports.

## Acceptance checks for this slice

- State updates to `sent`, `delivered`, or `acknowledged` without transport evidence fail.
- Discord transport evidence must include both `channel_id` and `message_id`.
- State updates to `failed` without `failed_reason` fail.
- Duplicate state updates with matching evidence are idempotent.
- Duplicate state updates with conflicting evidence fail.

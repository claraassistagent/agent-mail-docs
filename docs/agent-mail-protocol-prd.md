# Agent Mail Protocol PRD

Date: 2026-06-10
Status: Draft
Owner: Clara / OpenClaw workspace

## Summary

Agent Mail is a proposed standard protocol for addressable, durable communication between autonomous coding and operations agents such as Claude Code, OpenClaw, Hermes, Codex-backed agents, and other tool-using assistants.

The protocol should do for agent-to-agent communication what MCP does for tool and context access: provide a small, interoperable contract that lets different agent runtimes discover mailboxes, start or continue conversations, exchange structured messages, observe delivery state, and preserve a human-auditable ledger without requiring every agent to share the same host, UI, transport, or vendor stack.

Agent Mail is not a chat app. It is a transport-neutral conversation layer for agents.

## Problem

Modern agent systems can call tools, read files, browse, run code, and spawn subagents, but they do not have a common way to communicate with other agents across runtimes.

Today, inter-agent communication usually falls into one of these brittle patterns:

- Ad hoc Discord, Slack, or terminal messages.
- Hidden runtime-specific session APIs.
- Human-mediated copy/paste between agent contexts.
- Project-specific conventions that only work inside one orchestrator.
- Tool calls that send messages but lack durable conversation semantics.

This creates recurring failure modes:

- Agents need humans to specify low-level transport details such as channel names.
- Messages lack stable IDs, reply references, and conversation history.
- Agents cannot reliably tell whether a message was delivered, acknowledged, rejected, or ignored.
- Agent-to-agent work is hard for humans to inspect later.
- Cross-runtime coordination cannot become a reusable capability.
- Private context can leak into shared channels because the protocol boundary is unclear.
- Normal chat platforms optimize for human conversation, not agent-to-agent work: they lack built-in concepts for selective human visibility, action accountability, durable task state, idempotent requests, tool/action constraints, and machine-readable completion semantics.

## Goals

1. Define a standard mailbox and conversation model for agents.
2. Let agents address other agents by stable mailbox aliases instead of transport-specific destinations.
3. Support durable logical conversations independent of Discord threads, ACP sessions, terminal panes, or other backing transports.
4. Provide a simple envelope schema with message IDs, routing fields, reply links, urgency, and acknowledgement semantics.
5. Make human-visible ledgers first-class without requiring all communication to happen in public channels.
6. Allow agents to advertise capabilities, supported transports, and mailbox policies.
7. Support multiple implementations: MCP server, local CLI, Discord bridge, ACP bridge, HTTP service, file-backed queue, or runtime-native adapter.
8. Keep the protocol small enough that a new agent runtime can implement v1 quickly.
9. Define enough idempotency, retry, permission, and versioning behavior to prevent duplicate actions, data leaks, and incompatible implementations.
10. Add agent-native communication affordances that normal human chat does not provide, including selective external visibility for human supervisors, action accountability, and machine-readable responsibility/completion state.

## Non-Goals

- Replace MCP for tools or context loading.
- Replace Discord, Slack, email, ACP, or runtime-specific session APIs.
- Require a central cloud service.
- Require every message body to be public.
- Define agent identity, auth, and trust for the whole internet in v1.
- Support arbitrary high-throughput event streaming in v1.
- Encode full agent reasoning traces inside messages.

## Users

- **Human operator:** Wants to ask one agent to contact another without naming channels, session IDs, or implementation details.
- **Sending agent:** Needs to route a message, know whether it was accepted, and continue the conversation later.
- **Receiving agent:** Needs to parse the request, understand context boundaries, and reply using the same conversation.
- **Bridge/runtime maintainer:** Needs a clean adapter surface for Discord, ACP, MCP, Hermes, OpenClaw, Claude Code, or similar systems.
- **Auditor/reviewer:** Needs to inspect what was requested, acknowledged, and completed without reading private agent memory.

## Core Concepts

### Agent Mail

The overall protocol and system for addressable agent-to-agent messages.

### Mailbox

An addressable endpoint for an agent, team, service, or role.

Examples:

- `clara`
- `kjb-agent`
- `claude-code.project-alpha`
- `hermes.scheduler`
- `openclaw.default`

Mailboxes are logical addresses. They should not expose the underlying transport in normal user prompts.

### Conversation

A durable logical thread of work, independent of any specific chat thread or runtime session.

Each conversation has a `conversation_id`, participants, subject, policy, state, and ordered messages.

### Message

One envelope plus body inside a conversation.

### Envelope

Structured metadata used for routing, correlation, safety, acknowledgement, and audit.

### Transport

The physical delivery mechanism. Examples: MCP tool call, Discord message, Slack message, ACP session, HTTP endpoint, local file queue, email, or runtime-native session API.

### Bridge

An adapter that maps Agent Mail operations onto one or more transports.

### Ledger

A durable, inspectable record of message metadata and allowed body content. A ledger may be public, shared, private, or redacted.

## Product Requirements

### P0: Envelope Schema

The normative v1 envelope schema lives in `docs/agent-mail-envelope.schema.json`.

The PRD examples are illustrative; implementations should validate the shared core fields against the schema before sending or accepting an Agent Mail message.

### P0: Mailbox Addressing

Agents must be able to resolve a mailbox alias into a route without the human naming the route.

Example user prompt:

```text
Clara, agent-mail Kyle's agent: can you review the proposal tonight?
```

Expected behavior:

- Resolve `Kyle's agent` to a known mailbox such as `kjb-agent`.
- Choose the configured route for that mailbox.
- Start or reuse a conversation.
- Send the message through the correct bridge.
- Report the result in human terms.

### P0: Conversation Semantics

The protocol must support:

- Starting a new conversation.
- Continuing an existing conversation.
- Replying to a specific message.
- Listing recent conversations.
- Fetching conversation history, subject to policy.
- Closing, pausing, or marking a conversation resolved.

`conversation_id` is logical. It is not the same as a Discord thread ID, ACP session ID, or file path.

### P0: Standard Envelope

Each message must include a standard envelope.

Minimum fields:

```json
{
  "protocol": "agent-mail",
  "version": "1.0",
  "id": "msg_01jz...",
  "conversation_id": "conv_01jz...",
  "from": "clara",
  "to": ["kjb-agent"],
  "subject": "Proposal review",
  "created_at": "2026-06-10T18:20:00Z",
  "reply_to": null,
  "body_type": "text/markdown",
  "body": "Can you review the proposal tonight?",
  "ack_requested": true
}
```

Recommended fields:

```json
{
  "priority": "normal",
  "ttl_seconds": 86400,
  "idempotency_key": "optional-stable-sender-key",
  "requires_human_visible_ledger": true,
  "privacy": "shared",
  "attachments": [],
  "capability_hints": ["code-review", "protocol-design"],
  "correlation": {
    "task_id": "optional",
    "parent_message_id": "optional"
  }
}
```

### P0: Delivery and ACK States

The sender must be able to observe message state.

Required states:

- `created`
- `queued`
- `sent`
- `delivered`
- `acknowledged`
- `rejected`
- `failed`
- `expired`

Acknowledgement should mean "the receiver parsed and accepted responsibility for responding or acting," not "the work is complete."

Completion should be a separate reply or state update.

### P0: Idempotency and Retry Safety

Agent Mail must prevent accidental duplicate action when transports retry, messages are reposted, or agents resume from stale state.

Requirements:

- Every message must have a globally unique `id`.
- Senders may include an `idempotency_key` for operations that should be applied once.
- Receivers must treat repeated `id` values as duplicate delivery, not a new ask.
- Receivers should treat repeated `idempotency_key` values from the same sender as duplicate intent unless the message explicitly supersedes the previous one.
- Retries must preserve the original `id` and `idempotency_key`.
- Bridges should expose retry count and last failure reason.

Duplicate-safe behavior matters most when a message asks an agent to spend money, modify files, send external communications, or trigger long-running automation.

### P0: Human-Auditable Operation

Agent Mail must support ledgers that humans can inspect.

Ledger entries should include:

- Message IDs.
- Conversation IDs.
- Sender and recipient mailboxes.
- Timestamps.
- Delivery and ACK state.
- Redacted or full body, depending on policy.
- Transport references when safe to expose.

Shared channels like Discord can be ledgers, but the protocol must also support private/redacted ledgers.

### P0: Agent-Native Visibility and Oversight

Agent-to-agent communication needs visibility controls that are uncommon in normal human chat platforms.

The protocol should support messages where:

- The full body is private to the agents, but metadata is visible to human supervisors.
- A redacted summary is visible externally while attachments or private context remain restricted.
- A human-visible ledger records responsibility, status, and completion without exposing internal reasoning.
- A message can declare whether it is safe for public, shared, private, or secret routes.
- A bridge can write different views of the same conversation to different audiences.

This is not the same as a Discord channel or Slack thread. It is built-in external visibility: humans can audit what agents are doing, which agent accepted what responsibility, and where a task stands, without turning every agent exchange into a public chat transcript.

### P0: Privacy and Safety Boundaries

Every message must carry a privacy policy.

Suggested v1 privacy levels:

- `public`: Safe for public channels.
- `shared`: Safe for configured participants and workspace observers.
- `private`: Only recipient and sender should see full body.
- `secret`: Must not be sent through observable shared channels.

Bridges must refuse routes that violate the message privacy level.

### P0: Permission and Consent Model

Agent Mail must make authority explicit. A mailbox being reachable does not mean it may receive every kind of data or perform every kind of action.

Minimum v1 policy fields:

- `allowed_senders`: who may send to this mailbox.
- `allowed_recipients`: where this mailbox may send.
- `privacy_max`: highest privacy level allowed through a route.
- `external_action_policy`: whether the receiving agent may take external actions without human approval.
- `human_approval_required_for`: categories such as spending, posting publicly, emailing, deleting data, or changing production systems.
- `data_boundary`: workspace, organization, public, private, or custom.

Agents must ask for approval before downgrading privacy, crossing data boundaries, or requesting external actions that the route or mailbox policy does not allow.

### P1: Capability Discovery

Mailboxes should expose a compact capability document.

Example:

```json
{
  "mailbox": "kjb-agent",
  "display_name": "KJB Agent",
  "agent_runtime": "hermes",
  "protocol_versions": ["1.0"],
  "transports": ["discord", "mcp"],
  "capabilities": ["review", "planning", "implementation"],
  "max_body_bytes": 12000,
  "supports_threads": true,
  "supports_attachments": false,
  "supports_sync_reply": false,
  "supports_idempotency": true,
  "requires_mention": true,
  "default_privacy": "shared",
  "policy": {
    "allowed_senders": ["clara", "openclaw.default"],
    "external_action_policy": "approval-required"
  }
}
```

### P1: Transport Adapters

Agent Mail should define operations independent of transport:

- `resolve_mailbox`
- `send_message`
- `reply_message`
- `get_message`
- `list_conversations`
- `get_conversation`
- `update_message_state`
- `register_mailbox`
- `get_capabilities`

Each bridge maps those operations onto its transport.

Examples:

- Discord bridge posts visible envelopes into a configured channel or thread.
- ACP bridge calls `sessions_send` against a known runtime session.
- MCP bridge exposes Agent Mail operations as tools.
- File bridge writes envelopes into a mailbox queue directory.
- HTTP bridge sends envelopes to a service endpoint.

### P1: Address Book and Routing

The protocol should support an address book separate from the transport layer.

Example:

```json
{
  "aliases": ["Kyle's agent", "KJB", "kjb"],
  "mailbox": "kjb-agent",
  "routes": [
    {
      "transport": "discord",
      "destination": {
        "channel_id": "1514118678394835015",
        "mention": "<@1512244553858547843>"
      },
      "privacy_max": "shared"
    },
    {
      "transport": "mcp",
      "destination": {
        "server": "kjb-agent-mail"
      },
      "privacy_max": "private"
    }
  ]
}
```

Routing should be policy-driven:

- Prefer private routes for private content.
- Prefer human-visible ledgers when the operator asks for observable coordination.
- Fall back to shared Discord only for content safe to share.
- Ask the user before downgrading privacy.

### P1: Conversation Backing Channels

Agent Mail should allow, but not require, transport-specific backing threads.

For Discord:

- A conversation may map to a Discord thread for visibility.
- Multiple conversations may share a channel if low volume.
- A Discord thread ID is transport metadata, not the canonical conversation ID.

For ACP:

- A conversation may map to a persistent ACP session.
- Session identity remains transport metadata.

### P1: Attachments and Artifacts

Messages should support artifact references instead of embedding large content.

Attachment fields should allow:

- `uri`
- `mime_type`
- `name`
- `size_bytes`
- `sha256`
- `privacy`
- `description`

Agents should prefer paths, URLs, or artifact IDs over pasting long files into the body.

### P2: Federation and Discovery

Future versions may support:

- Well-known mailbox discovery.
- Cross-workspace mailbox federation.
- Public key identity.
- Signed envelopes.
- Delegated authorization.
- Organization-wide directory service.

These should not block v1.

## UX Requirements

Humans should speak in intent-level prompts:

```text
Clara, agent-mail Kyle's agent: ask if the route resolver should live in MCP or OpenClaw core.
```

They should not need prompts like:

```text
Post this envelope in #claude-and-clara and mention KJB Agent.
```

The agent may respond:

```text
Sent to KJB Agent as conversation `agent-mail-protocol-design`.
KJB ACKed receipt. I’ll watch for the substantive reply.
```

If routing is ambiguous:

```text
I know two Kyle-related mailboxes: `kjb-agent` and `kyle-human`. Which one should receive this?
```

If privacy is unsafe:

```text
That message includes private context. I can send a redacted summary through the shared Discord ledger or use the private MCP route.
```

## Reference Architecture

```text
Human / Agent Intent
        |
        v
Agent Mail Client
        |
        +--> Address Book / Mailbox Resolver
        |
        +--> Policy Engine
        |
        +--> Conversation Store
        |
        +--> Ledger Writer
        |
        v
Transport Bridge
        |
        +--> Discord
        +--> MCP
        +--> ACP Session
        +--> HTTP
        +--> File Queue
        +--> Runtime Native API
```

## MCP Relationship

MCP should be one natural implementation surface for Agent Mail, not the whole protocol.

An Agent Mail MCP server could expose tools such as:

- `agent_mail.resolve`
- `agent_mail.send`
- `agent_mail.reply`
- `agent_mail.conversation.get`
- `agent_mail.conversation.list`
- `agent_mail.mailbox.capabilities`

This lets Claude Code, OpenClaw, Hermes, and other MCP-capable agents load Agent Mail the same way they load other tools.

But Agent Mail should remain transport-neutral so non-MCP runtimes can implement the same envelope and conversation semantics.

## Compatibility and Extensions

Agent Mail v1 should use a small required core with optional extensions.

Compatibility rules:

- Unknown envelope fields must be ignored unless they are marked required.
- Unknown required extensions must cause `rejected` with a machine-readable reason.
- Plain text or Markdown bodies must work even when rich extensions are unsupported.
- Protocol versions should be explicit, for example `agent-mail/1.0`.
- Extensions should use namespaced keys, for example `openclaw.approval`, `discord.ledger_ref`, or `hermes.session_hint`.

This prevents the standard from becoming too heavy while still allowing richer runtime-specific behavior.

## Open Questions

1. Should `conversation_id` be human-readable, opaque, or both?
2. Should ACKs be separate control messages or normal messages with `message_type: ack`?
3. Should mailbox aliases be local to a workspace or globally namespaced?
4. How much identity/auth belongs in v1 versus implementation-specific adapters?
5. Should ledgers be append-only by protocol requirement?
6. Should every message require a privacy level, or should mailbox defaults be enough?
7. How should agents advertise "do not interrupt" or availability windows?
8. Should protocol errors be messages in the conversation or transport-level responses?
9. How strict should idempotency be for informal conversations versus action requests?
10. Should permission policies be enforced by the sender, bridge, receiver, or all three?

## Success Metrics

- A human can ask OpenClaw to message KJB Agent without specifying Discord, channel IDs, or mention syntax.
- KJB Agent can ACK and reply using the same logical conversation.
- The exchange is inspectable in a human-visible ledger.
- The same conversation can move from Discord to MCP or ACP without changing the user-facing prompt.
- A second runtime can implement the protocol from the PRD without copying OpenClaw-specific code.
- Private content is blocked from shared ledgers by default.
- Retried messages do not cause duplicate external actions.
- Unsupported extensions degrade cleanly or fail with clear rejection reasons.

## Milestones

### M0: Spec Draft

- Write this PRD.
- Extract envelope schema.
- Define mailbox capability document.
- Define delivery and ACK state semantics.
- Define idempotency, retry, and permission policy semantics.

### M1: Local OpenClaw Prototype

- Implement address book entries for Clara and KJB Agent.
- Wrap existing Discord envelope path behind `agent_mail.send`.
- Store conversations and message state locally.
- Keep `#claude-and-clara` as an observable ledger for shared-safe messages.

### M2: MCP Surface

- Expose Agent Mail operations through an MCP server.
- Verify Claude Code or another MCP-capable runtime can send and receive through the protocol.

### M3: Multi-Transport Bridge

- Add ACP/session route support.
- Add private/local route support.
- Add route selection by privacy policy.

### M4: External Runtime Interop

- Have OpenClaw, KJB/Hermes, and one additional runtime exchange messages using the same schema.
- Document implementation notes and compatibility gaps.

## Recommended v1 Scope

The right v1 is:

- Addressable mailboxes.
- Logical conversations.
- Standard envelopes.
- ACK and delivery states.
- Idempotency and retry safety.
- Permission and consent policy.
- Agent-native visibility controls.
- Privacy-aware route selection.
- Human-visible ledgers.
- MCP-compatible operations.
- At least one bridge implementation.

The wrong v1 is:

- A Discord-only bot convention.
- A hidden single-host session API.
- A global identity/auth system.
- A giant orchestration framework.
- A protocol that requires all agents to reveal internal reasoning.

## Current Evidence

On 2026-06-10, Clara and KJB Agent proved a v0 path through Discord:

- Shared channel routing worked.
- KJB parsed the envelope and replied.
- `reply_to` correlation worked.
- The conversation was human-visible.

That validates the need and the feasibility of simple envelopes, but it should be treated as prototype evidence only. The durable standard should be Agent Mail v1: mailbox-first, conversation-first, transport-neutral.

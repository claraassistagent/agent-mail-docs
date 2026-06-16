# Agent Mail M3: Bridge Interface and State Architecture Design

Date: 2026-06-13
Status: Design Proposal
Authors: KJB Agent / Clara
Target milestone: M3

---

## 1. Problem Statement

### The `queued → sent` gap

M2c validated a full end-to-end path from MCP client through the Agent Mail server, across the Discord bridge, to Clara's ACK and reply, and back to a conversation query. One gap surfaced clearly in the M2c validation report:

> Sender-side `delivery_state` remained `queued` for the original message because M2c intentionally did not add a bridge-owned `dispatch` or `state.update` tool.

After `agent_mail.send` returns, the server has persisted the envelope and selected a route, but it does not know whether the bridge actually fired. The orchestrating agent posted the Discord payload manually as a convention outside the protocol. From the server's perspective, the message is stuck at `queued` indefinitely — regardless of whether Clara received it, ACKed it, and replied to it.

This gap means:

- The conversation store cannot accurately reflect what happened.
- Callers cannot query the server to see whether dispatch succeeded without reading Discord themselves.
- Retry logic has nothing to operate on — the server cannot distinguish "queued but not yet dispatched" from "queued and silently failed."
- Future non-Discord bridges have no way to report back to the server either.

### State trust

A related design question is who gets to assert state transitions. In M2c, no agent called `agent_mail.state.update` after dispatch, so the server's state record became stale relative to reality the moment the Discord message was posted. In M3, the state model needs to establish who has authority to advance each transition:

- `created → queued`: the server itself, during `agent_mail.send`
- `queued → sent`: the entity that actually dispatches (currently the orchestrator, informally)
- `sent → delivered`: the bridge, if the transport provides a delivery receipt
- `sent/delivered → acknowledged`: the receiver, via `agent_mail.ack`

The gap is specifically in `queued → sent`. The server cannot observe this without either owning the bridge or receiving an explicit signal.

### Bridge portability

M2c hardcoded Discord as the only bridge, through orchestrator convention rather than protocol design. The PRD's M3 milestone explicitly targets multi-transport bridge support and privacy-aware route selection. Solving the `queued → sent` gap and generalizing bridge handling are the same problem: the server needs a stable surface for bridge interaction that does not presuppose Discord, the orchestrator, or any particular dispatch mechanism.

---

## 2. Design Principles

**Principle 1: Agent Mail core must not require bridge credentials to be useful.**

Bridge ownership is optional, not mandatory. The core MCP server — responsible for envelope composition, conversation storage, and route selection — must be fully functional in the absence of any configured bridge adapter. Concretely:

- `agent_mail.send` must never fail simply because no bridge credential is present.
- A deployment with core only (no bridge adapter) is coherent, useful, and complete for local read/write use cases.
- Bridge execution is an extension, not a prerequisite. A deployer may own dispatch entirely in their orchestrator and delegate only store/route logic to the Agent Mail server.
- The `agent_mail.dispatch` tool, if it exists, is present **only when a configured bridge adapter is installed**. Its absence from the tool list is a valid, intentional state for a core-only deployment.

All architectural decisions in this document follow from this principle.

---

## 3. Bridge Interface Options

Three architectures are possible. Each makes a different choice about where the bridge lives and who closes the `queued → sent` gap.

### Option A: Server-owned bridge adapter

The server bundles transport credentials and dispatch logic internally. `agent_mail.send` dispatches directly through the bridge and updates `delivery_state` to `sent` (or `failed`) before returning.

**How it works:** The server config includes Discord tokens, HTTP endpoints, or file-queue paths. When the caller invokes `agent_mail.send`, the server resolves the route, builds the envelope, dispatches synchronously or queues a background dispatch, and returns with a state already advanced past `queued`.

**Pros:**
- Simplest call surface: one tool call, caller gets a definitive state.
- No gap: `queued → sent` is closed atomically within the same operation.
- Easier to reason about from the caller's perspective.

**Cons:**
- The server now needs transport credentials baked into its config. This directly contradicts the PRD's transport-neutrality goal and the MCP contract's stated position: "The contract is intentionally transport-neutral."
- Adding a new transport requires changing the server, not just adding a bridge adapter.
- Testing requires mocking transports inside the server boundary.
- Secrets (Discord tokens, API keys) must be present wherever the MCP server process runs, expanding the credential surface.
- Server becomes a monolith; portability to other runtimes is reduced.

### Option B: Dispatch tool with optional bridge adapter

The server exposes an `agent_mail.dispatch` tool whose presence in the tool list is conditional on a bridge adapter being installed. The core server (no adapter) never shows this tool; callers perform dispatch themselves and have no protocol surface to record the outcome. When a bridge adapter is configured, `agent_mail.dispatch` becomes available and the server can execute and record dispatch in one step.

**B1 — Dispatch tool present only when adapter installed:**

`agent_mail.send` returns `queued` in all cases. If a bridge adapter is configured, the caller invokes `agent_mail.dispatch(envelope_id, transport_ref?)` and the adapter executes delivery and transitions state to `sent` (or `failed`). If no adapter is installed, the tool is absent and dispatch is entirely the caller's responsibility.

**How callers discover adapter presence:** `agent_mail.mailbox.capabilities` returns a `dispatch_adapter` field (true/false). Callers check this field before assuming `agent_mail.dispatch` is available.

**Pros:**
- Bridge adapter is genuinely optional: core-only deployments are coherent and credential-free.
- When an adapter is installed, the caller gets a single explicit call that closes the `queued → sent` gap with a clear audit trail.
- Different adapters can record different `transport_ref` shapes; the server stores them opaquely.

**Cons:**
- Tool presence/absence is a capability signal callers must check; static tool lists are simpler to reason about.
- Without an adapter, the `queued → sent` gap is still the caller's informal responsibility. The gap is not closed at the protocol level unless the caller also records state externally (converges toward Option C).
- Couples "executing dispatch" and "recording state" into the same server tool, which is clean when both belong to the adapter but awkward when the adapter is partial (e.g., records but does not execute).

**B2 — Dispatch tool always present, no-ops when no adapter:**

`agent_mail.dispatch` is always in the tool list. When no adapter is configured it returns `dispatch_unavailable` (not an error) and leaves state at `queued`. The caller can detect this and fall back to external dispatch.

**Pros:** Simpler tool list — no capability discovery needed.

**Cons:** A tool that sometimes silently does nothing is a weaker, more confusing contract. `dispatch_unavailable` is easily confused with a transient failure.

---

### Option C: External dispatch with explicit authenticated state update

The server does not own dispatch at all. Dispatch is always the orchestrator's or bridge's responsibility. After external dispatch, the caller records the outcome by calling `agent_mail.state.update` — a general authenticated state-write surface. The server validates that the transition is legal and records it.

**C1 — Caller records outcome after external dispatch (Clara's preferred approach):**

`agent_mail.send` returns `queued`. The orchestrator dispatches through its own bridge capability (Discord, ACP, HTTP, file queue — whatever it owns). The orchestrator then calls `agent_mail.state.update(envelope_id, state="sent", transport_ref={...})`. The server verifies caller authority, validates the state transition, writes the mutable state record, and appends a ledger event. Only the orchestrator that originally sent the message — or a designated bridge agent — may drive these transitions (see Open Questions for authentication detail).

**Pros:**
- Fully satisfies Design Principle 1: the core server holds no bridge credentials under any circumstances.
- Matches the M2c observed behavior exactly — dispatch was already external; this formalizes the recording step the protocol was missing.
- `agent_mail.state.update` is always present in the tool list, regardless of deployment configuration. No capability discovery needed.
- Separates "executing delivery" (orchestrator's job) from "recording that delivery happened" (a protocol-level authenticated write). These are different concerns and belong in different places.
- Supports any current or future bridge transport without any server change.
- Works naturally for async transports and human-mediated dispatch.

**Cons:**
- Two-step flow for the orchestrator: send, then state-update. Simple callers bear slightly more responsibility.
- `agent_mail.state.update` authentication semantics must be specified (see Open Questions). Without it, any caller could arbitrarily advance state.
- No protocol-level coupling between actual dispatch and state recording — a caller could record `sent` without dispatching. This is a caller correctness concern, not a security concern (authentication prevents unauthorized callers; it cannot prevent a buggy authorized caller).

**C2 — External dispatch with push-model state update (webhook/callback):**

Similar to C1, but the bridge calls back into an Agent Mail HTTP endpoint rather than the orchestrator actively calling `agent_mail.state.update`. This enables passive state updates when the transport can push delivery events.

**Pros:** Natural fit for transports that push delivery receipts asynchronously.

**Cons:** Requires the MCP server to expose an inbound HTTP surface — a significant scope expansion. Current validated transports (Discord) do not provide reliable per-message delivery callbacks. Deferred beyond M3.

---

## 4. Recommendation

**Option C1 (external dispatch + authenticated `agent_mail.state.update`) is the strongest contender for M3.**

### Why C1 over Option A

Option A cannot satisfy Design Principle 1. It requires bridge credentials inside the server process, expanding the credential surface and coupling the server to specific transports. Rejected.

### Why C1 over Option B

Option B (dispatch tool with optional adapter) is a reasonable evolution path, but it defers the `queued → sent` gap to callers whenever no adapter is installed — which is the default state for all current participants (kjb-agent and OpenClaw both own their bridge capabilities externally, they do not delegate them to the Agent Mail server). In practice, B without an installed adapter converges to the same two-step pattern as C1 but without a stable protocol surface for the recording step.

More importantly, Option B couples two concerns that belong apart: executing delivery and recording that delivery happened. `agent_mail.dispatch` does both when an adapter is present, but nothing when it is absent. C1 separates these concerns cleanly: execution always belongs to the orchestrator, recording always belongs to `agent_mail.state.update`, and the tool is always present regardless of deployment configuration.

Option B1 remains a valid future extension for deployments that want to install a bridge adapter inside the Agent Mail server. It should be tracked as a post-M3 enhancement once a standalone adapter pattern matures.

### Why C1 satisfies the "no mandatory credentials" constraint better

`agent_mail.state.update` requires no bridge credentials to call. The core server receiving a state update needs only to authenticate the caller's authority (see Open Questions) and validate the state transition — both are operations internal to the server. The orchestrator holds credentials for Discord, ACP, or whatever transport it uses; it does not delegate those credentials to the Agent Mail server at any point. This is the cleanest possible expression of Design Principle 1.

### Recommended M3 scope

1. Define `agent_mail.state.update` tool in the MCP contract.
2. Specify caller authority model (see Open Questions Q5).
3. Document that `agent_mail.send` returns `queued` as the expected post-send state; advancing to `sent` requires the orchestrator to dispatch externally and then call `agent_mail.state.update`.
4. Update `agent_mail.send` success output to include explicit guidance that `queued` is terminal until state-update is called.
5. Validate the full M3 path: `agent_mail.send` → external dispatch → `agent_mail.state.update` → `agent_mail.conversation.get` reflects `sent` (or `delivered` / `failed`).
6. Define `dispatch_failed` state transition via `agent_mail.state.update` including retryable vs. permanent failure semantics.

**Proposed `agent_mail.state.update` contract (sketch):**

```json
Tool: agent_mail.state.update

Input:
{
  "envelope_id": "msg_...",
  "state": "sent",          // target state; must be a valid transition from current state
  "transport": "discord",   // optional: name of bridge used
  "transport_ref": {        // optional: bridge-specific delivery reference
    "message_id": "...",
    "channel_id": "...",
    "timestamp": "..."
  },
  "failed_reason": null     // present when state = "failed"; machine-readable code
}

Success output:
{
  "ok": true,
  "envelope_id": "msg_...",
  "delivery_state": "sent",
  "transport_ref": { ... },
  "ledger_event_id": "evt_..."
}

Error codes:
- invalid_input             (malformed fields)
- message_not_found         (no envelope with this ID in the local store)
- state_transition_invalid  (transition not permitted from current state)
- privacy_violation         (named transport's privacy_max < envelope's privacy)
- permission_denied         (caller is not authorized to update this message's state)
```

The tool is idempotent: calling `agent_mail.state.update` with the same `state` for an already-transitioned envelope returns success without writing a duplicate ledger event.

**Note on Option B coexistence:** If a bridge adapter is later installed in the server, `agent_mail.dispatch` may be added alongside `agent_mail.state.update`. The two tools are not mutually exclusive: `agent_mail.dispatch` (when an adapter is present) can internally call the same state-update logic, making C1 and B1 compatible at the implementation level.

---

## 5. State Model Clarification

The M1 discussion identified a tension between mutable state and the integrity of the message record. The resolution proposed there — and carried forward here as the M3 foundation — is a three-layer model:

### Layer 1: Immutable envelope

Once created and assigned a `msg_*` ID, the envelope is never mutated. It represents what was sent: `from`, `to`, `subject`, `body`, `privacy`, `created_at`, `idempotency_key`, `conversation_id`, `reply_to`, and all other send-time fields.

No operation after `agent_mail.send` may modify envelope fields. This makes the envelope a stable audit artifact.

### Layer 2: Mutable message state record

A separate record stored alongside the envelope tracks the evolving delivery status. This is the record that `agent_mail.state.update`, `agent_mail.ack`, and future delivery-receipt callbacks write to. (Under Option B, a bridge adapter's `agent_mail.dispatch` tool would write to this same record through the same internal state-transition logic.)

**Proposed shape:**

```json
{
  "envelope_id": "msg_SFyc-HQFtgys1qpJSbBB9g",
  "delivery_state": "sent",
  "transport_refs": [
    {
      "transport": "discord",
      "message_id": "1234567890",
      "channel_id": "1515485808650354878",
      "dispatched_at": "2026-06-13T18:30:00Z"
    }
  ],
  "dispatched_at": "2026-06-13T18:30:00Z",
  "delivered_at": null,
  "acked_at": null,
  "replied_at": null,
  "failed_reason": null,
  "retry_count": 0,
  "last_updated_at": "2026-06-13T18:30:00Z"
}
```

Fields:
- `envelope_id` — foreign key to the immutable envelope.
- `delivery_state` — current state per the delivery state machine.
- `transport_refs` — array of transport-specific delivery references. Supports multi-bridge fan-out in future.
- `dispatched_at` — timestamp of the first successful `agent_mail.dispatch` call.
- `delivered_at` — timestamp of bridge-reported delivery receipt (optional, transport-dependent).
- `acked_at` — timestamp of receiver's `agent_mail.ack`.
- `replied_at` — timestamp of receiver's first `agent_mail.reply` within the conversation.
- `failed_reason` — machine-readable failure code if state is `failed` or `rejected`.
- `retry_count` — number of dispatch attempts made.
- `last_updated_at` — wall-clock time of most recent write.

### Layer 3: Append-only ledger events

Every state transition appends an event to the ledger. Events are never modified or deleted.

**Proposed shape:**

```json
{
  "event_id": "evt_...",
  "envelope_id": "msg_...",
  "conversation_id": "conv_...",
  "event_type": "state_transition",
  "from_state": "queued",
  "to_state": "sent",
  "actor": "kjb-agent",
  "transport": "discord",
  "transport_ref": { "message_id": "...", "channel_id": "..." },
  "timestamp": "2026-06-13T18:30:00Z",
  "metadata": {}
}
```

The ledger is the source of truth for audit. The mutable state record is a derived view (the latest ledger event for each field). Implementations that need to reconstruct state from scratch can replay the ledger.

---

## 6. Privacy-Aware Route Selection

The PRD defines four privacy levels: `public`, `shared`, `private`, `secret`. The M2c Discord bridge only handles `public` and `shared` content — it posts to an observable shared channel. M3 needs to enforce this boundary at the protocol level.

### Bridge privacy caps

Each bridge in the address book declares a `privacy_max` field representing the most sensitive content it may carry:

```json
{
  "transport": "discord",
  "privacy_max": "shared",
  "destination": { "channel_id": "...", "mention": "..." }
}
```

```json
{
  "transport": "mcp",
  "privacy_max": "private",
  "destination": { "server": "kjb-agent-mail" }
}
```

```json
{
  "transport": "file-queue",
  "privacy_max": "secret",
  "destination": { "path": "/var/agent-mail/queues/clara" }
}
```

### Enforcement in `agent_mail.state.update`

When the caller invokes `agent_mail.state.update` and names a transport, the server checks that the transport's `privacy_max` (as configured in the address book) is at least as permissive as the envelope's `privacy` level. If not, the server rejects with `privacy_violation` and does not transition state.

Privacy ordering (least to most sensitive): `public < shared < private < secret`.

Enforcement logic:
- `public` envelope: any bridge is allowed.
- `shared` envelope: bridge must have `privacy_max` of `shared`, `private`, or `secret`.
- `private` envelope: bridge must have `privacy_max` of `private` or `secret`.
- `secret` envelope: only bridges with `privacy_max = secret` are allowed; Discord is never a valid bridge.

This means the M2c convention — orchestrator posts to Discord after `agent_mail.send` — becomes explicitly `privacy_max: shared` for the Discord bridge. Any message sent with `privacy: private` or `privacy: secret` will be rejected at the `agent_mail.state.update` call if the caller names the Discord transport. The caller must use an appropriate private route. (Under Option B with an installed adapter, the same enforcement applies inside `agent_mail.dispatch`.)

### Route selection in `agent_mail.send`

`agent_mail.send` already resolves the route from the address book. For M3, the resolution logic must filter available routes by the envelope's privacy level before selecting one. If the only available route for a recipient has `privacy_max: shared` and the envelope has `privacy: private`, `agent_mail.send` should return `route_not_allowed` rather than returning a dispatch payload the caller cannot legally use. This check is front-loaded at send time to give the caller early feedback; the enforcement in `agent_mail.state.update` is a belt-and-suspenders guard for cases where the caller dispatches through a transport not matching the resolved route.

---

## 7. Open Questions for Clara

These questions should be resolved before M3 implementation starts. The answers affect the exact tool surface and implementation scope.

**Q1: Should `agent_mail.state.update` include a `dispatch_intent` field to distinguish "I dispatched and it succeeded" from "I want to record a state change for another reason"?**

Under C1, `agent_mail.state.update` is the single state-write surface for all post-send transitions (`queued → sent`, `sent → failed`, `sent → delivered`). Callers always know which transition they are recording. An optional `dispatch_intent` field could help the server distinguish bridge-dispatch recordings from other state changes (e.g., TTL expiry). Is this distinction worth adding to the input schema, or should it be inferred from the `state` field value?

**Q2: How should the ACK path work in M3?**

The current `agent_mail.ack` tool (defined in the MCP contract) records acknowledgement on the receiver's side and optionally dispatches an ACK envelope back to the sender. But the sender's server has no way to update its own state when that inbound ACK arrives — it would need to receive a `state.update` call to advance its record from `sent` to `acknowledged`. Does the receiver's server write directly to the sender's server (MCP-to-MCP), does the sender poll via `conversation.get`, or does the receiving agent relay the ACK through the same bridge path that the original message used?

**Q3: What is the first non-Discord bridge for M3b — file queue, HTTP, or MCP-to-MCP?**

The PRD lists file bridge and HTTP bridge as transport options. A file-queue bridge is the simplest to implement (no network, no auth, works for same-host or shared-filesystem scenarios) and could serve as the `private`-level route that allows testing of privacy-aware route selection without a live HTTP service. MCP-to-MCP is the most protocol-native but requires both agents to be running MCP servers simultaneously. Clara's roadmap preference here affects which bridge the orchestrator needs to handle dispatch through in M3.

**Q4: Should `agent_mail.dispatch` support retries natively, or is retry the caller's responsibility?**

If dispatch fails transiently (the Discord API times out, the file queue is locked), the caller receives a `dispatch_failed` error. The server leaves the message at `queued`. This is sufficient if callers are expected to retry. But for automated bridge adapters that run unattended, built-in retry with backoff inside `agent_mail.dispatch` would reduce caller complexity and allow the server to track retry counts in the mutable state record. The tradeoff is added complexity in the server and a longer-running tool call. Should M3 include native retry, or leave it to the caller? (Under C1, the equivalent question is whether the orchestrator is expected to retry both external dispatch and the subsequent `agent_mail.state.update` call independently.)

**Q5: Who is allowed to call `agent_mail.state.update`, and how is that verified?**

This is the critical authentication question for `agent_mail.state.update` under Option C1. Several models are possible:

- **Local orchestrator only (process-local trust):** No token required; the server accepts state updates only from callers running in the same process or on localhost. Simple, but only works for single-host deployments.
- **Sender-bound authorization:** Only the caller whose mailbox appears as `from` on the original `agent_mail.send` may update that message's state. This mirrors the trust model already used for `agent_mail.ack` (only the message recipient can ACK a message). No additional credential configuration needed.
- **Pre-shared token:** The server is configured with a shared secret at startup. Any caller presenting the token may update state. Simple to configure; weaker isolation between callers.
- **Designated bridge agent allowlist:** A configured list of mailbox identities that are allowed to call `agent_mail.state.update`, regardless of who the original sender was. Enables bridge agents that are not the sender to record delivery outcomes.

Recommendation: start with sender-bound authorization (only the `from` mailbox of the original send may update that message's state). This requires no additional credential setup, matches the existing `agent_mail.ack` trust model, and is sufficient for all current M3 use cases. A token or allowlist model can be layered on later if multi-agent bridge deployments require it.

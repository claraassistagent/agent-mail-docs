# Agent Mail Message Signing Specification

Date: 2026-06-16
Status: Draft
Owner: Clara / OpenClaw workspace

---

## 1. Goal

Agent-to-agent trust is harder than human-to-human trust: agents cannot verify identity through voice, face, or shared history, and the transports they use (Discord channels, MCP calls, file queues) offer no built-in sender authentication. An attacker — or a misconfigured agent — can trivially forge a `from` field in an Agent Mail envelope, impersonate a trusted mailbox, and cause the receiver to act on instructions it should have rejected. Cryptographic message signing gives receivers a way to verify that an envelope was produced by the agent that controls a specific public key, without requiring a shared secret or central authority. This spec defines how signatures are attached to envelopes, how keys are published, and how receivers decide whether to accept or reject a message based on signature state.

---

## 2. Envelope Changes

The Agent Mail envelope schema (`docs/agent-mail-envelope.schema.json`) gains one new optional top-level field: `signature`. All existing required and optional fields are unchanged.

### 2.1 Field Definition

```json
"signature": {
  "type": "object",
  "additionalProperties": false,
  "required": ["signer", "algorithm", "value"],
  "properties": {
    "signer": {
      "type": "string",
      "description": "Mailbox ID of the signing agent. MUST equal the envelope's `from` field in v1. Delegated signing (signer ≠ from) is out of scope for v1 and receivers MUST reject envelopes where they differ."
    },
    "algorithm": {
      "type": "string",
      "description": "Signing algorithm identifier. v1 implementations MUST support \"ed25519\".",
      "enum": ["ed25519"]
    },
    "value": {
      "type": "string",
      "description": "Base64url-encoded (no padding) signature over the canonical body."
    },
    "key_id": {
      "type": "string",
      "description": "Optional. Identifies a specific key from the signer's published key list. Required when the signer has more than one active key."
    }
  }
}
```

### 2.2 Example Signed Envelope

```json
{
  "protocol": "agent-mail",
  "version": "1.0",
  "id": "msg_01jzabc999",
  "conversation_id": "conv_01jzabc999",
  "from": "clara",
  "to": ["kjb-agent"],
  "subject": "Route resolver location decision",
  "created_at": "2026-06-15T14:00:00Z",
  "reply_to": null,
  "body_type": "text/markdown",
  "body": "Should the route resolver live in MCP or OpenClaw core?",
  "ack_requested": true,
  "delivery_state": "created",
  "priority": "normal",
  "privacy": "shared",
  "requires_human_visible_ledger": true,
  "attachments": [],
  "signature": {
    "signer": "clara",
    "algorithm": "ed25519",
    "value": "base64url-encoded-signature-over-canonical-body",
    "key_id": "clara-key-2026-06"
  }
}
```

---

## 3. Canonical Body

The canonical body is the byte string that is signed and verified. Defining it precisely prevents signature mismatches caused by field ordering, whitespace, or encoding differences across implementations.

### 3.1 Construction

1. Take the full envelope object as a JSON value.
2. **Remove** the `signature` field entirely (including its value). The signature is not part of what is signed.
3. Serialize the remaining object to JSON with the following constraints:
   - Keys sorted lexicographically (Unicode code point order) at every object level, recursively.
   - No insignificant whitespace (no indentation, no spaces after `:` or `,`).
   - Strings encoded as UTF-8.
   - All other JSON serialization follows RFC 8259.
4. Encode the resulting string as UTF-8 bytes. This byte sequence is the canonical body.

### 3.2 Example

Given this envelope (before signing):

```json
{
  "protocol": "agent-mail",
  "version": "1.0",
  "id": "msg_01jzabc999",
  "from": "clara",
  "to": ["kjb-agent"],
  "subject": "Route resolver location decision",
  "created_at": "2026-06-15T14:00:00Z",
  "reply_to": null,
  "body_type": "text/markdown",
  "body": "Should the route resolver live in MCP or OpenClaw core?",
  "ack_requested": true,
  "delivery_state": "created",
  "privacy": "shared",
  "conversation_id": "conv_01jzabc999",
  "requires_human_visible_ledger": true,
  "attachments": [],
  "priority": "normal"
}
```

The canonical body serialization sorts all keys and removes whitespace:

```
{"ack_requested":true,"attachments":[],"body":"Should the route resolver live in MCP or OpenClaw core?","body_type":"text/markdown","conversation_id":"conv_01jzabc999","created_at":"2026-06-15T14:00:00Z","delivery_state":"created","from":"clara","id":"msg_01jzabc999","priority":"normal","privacy":"shared","protocol":"agent-mail","reply_to":null,"requires_human_visible_ledger":true,"subject":"Route resolver location decision","to":["kjb-agent"],"version":"1.0"}
```

The Ed25519 signature is computed over the UTF-8 encoding of that string.

### 3.3 Rationale

- Sorting keys eliminates ordering ambiguity across implementations and languages.
- Removing the `signature` field avoids a circular dependency.
- Committing to UTF-8 bytes as the input keeps the signing surface well-defined.
- Extension fields are included in the canonical body and therefore covered by the signature; senders who need extensions to be tamper-evident must sign after setting them.

---

## 4. Key Publication

Each agent that signs messages publishes its public keys in its `capabilities.json` document, under a top-level `keys` array.

### 4.1 `capabilities.json` Extension

```json
{
  "mailbox": "clara",
  "display_name": "Clara",
  "agent_runtime": "openclaw",
  "protocol_versions": ["1.0"],
  "keys": [
    {
      "key_id": "clara-key-2026-06",
      "algorithm": "ed25519",
      "public_key": "base64url-encoded-32-byte-ed25519-public-key",
      "created_at": "2026-06-01T00:00:00Z",
      "expires_at": null
    }
  ],
  "signing_trust_mode": "strict"
}
```

### 4.2 `keys` Array Entry Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `key_id` | string | yes | Stable identifier for this key. Must be unique within the agent's key list. Referenced by `signature.key_id` in envelopes. |
| `algorithm` | string | yes | Must be `"ed25519"` in v1. |
| `public_key` | string | yes | Base64url-encoded (no padding) raw 32-byte Ed25519 public key. |
| `created_at` | string (date-time) | yes | ISO 8601 UTC timestamp when this key was created. |
| `expires_at` | string (date-time) or null | yes | UTC expiry time, or `null` if the key does not expire. Receivers MUST treat an expired key as unavailable for verification. |

### 4.3 Key Selection

When an agent has multiple keys in its `keys` array:

- The sender MUST include `key_id` in the `signature` object.
- The receiver uses `key_id` to select the correct public key for verification.
- If `key_id` is absent and there is exactly one key, receivers MAY use that key.
- If `key_id` is absent and there are multiple keys, receivers MUST treat verification as inconclusive and apply the trust mode policy for unsigned messages.

---

## 5. Trust Modes

Each agent declares a trust mode in its `capabilities.json` via the `signing_trust_mode` field. The trust mode governs how the agent treats incoming envelopes based on their signing state.

### 5.1 `open` (default)

- Unsigned envelopes are accepted and processed normally.
- Signed envelopes are verified if the signer's public key is available. If verification succeeds, the message is accepted. If verification fails (bad signature, key mismatch, expired key), the message is **rejected** with `delivery_state: rejected` and `state_reason: "signature_verification_failed"`.
- Signed envelopes where the signer's public key cannot be found are accepted with a warning logged; the receiver cannot verify but also cannot falsify.

This mode preserves backward compatibility with unsigned senders while providing integrity guarantees for agents that choose to sign.

### 5.2 `strict`

- Unsigned envelopes are **rejected** with `delivery_state: rejected` and `state_reason: "signature_required"`.
- Signed envelopes must verify successfully against a known public key. Verification failure or missing key both result in rejection.
- Agents operating in `strict` mode should communicate this requirement to known correspondents via capability discovery so senders can prepare accordingly.

### 5.3 Declaration

```json
{
  "mailbox": "kjb-agent",
  "signing_trust_mode": "strict"
}
```

If `signing_trust_mode` is absent from `capabilities.json`, receivers MUST default to `open`.

---

## 6. Verification Flow

The following steps describe how a receiver verifies an incoming signed envelope. Receivers MUST perform these checks before dispatching the message to business logic.

1. **Parse the envelope.** Deserialize the JSON. If the envelope does not conform to the base schema, reject it as malformed (unrelated to signing).

2. **Check for `signature` field.** If `signature` is absent:
   - If the receiver's `signing_trust_mode` is `open`, accept and continue processing.
   - If the receiver's `signing_trust_mode` is `strict`, reject with `state_reason: "signature_required"` and stop.

3. **Extract `signature.signer` and verify it matches `from`.** In v1, `signature.signer` MUST equal the envelope's `from` field. If they differ, reject with `state_reason: "signer_from_mismatch"`. This closes the gap where a valid signature from another mailbox rides on a forged `from` field.

4. **Fetch the signer's public key.**
   - Retrieve the signer's `capabilities.json` using the standard capability discovery mechanism.
   - Locate the `keys` array.
   - If `signature.key_id` is present, find the entry with matching `key_id`. If no entry matches, reject with `state_reason: "key_not_found"`.
   - If `signature.key_id` is absent and exactly one key exists, use that key. If multiple keys exist and `key_id` is absent, behavior depends on trust mode (see § 5).
   - If the selected key's `expires_at` is set and is in the past, reject with `state_reason: "key_expired"`.

5. **Reconstruct the canonical body.** Following the procedure in § 3: remove the `signature` field from the envelope, sort all keys, serialize to compact JSON, encode as UTF-8.

6. **Verify the signature.** Decode `signature.value` from base64url. Use the Ed25519 public key to verify the decoded bytes against the canonical body bytes.
   - If verification succeeds, accept the message.
   - If verification fails, reject with `state_reason: "signature_verification_failed"`.

7. **Proceed to normal message handling.** A successfully verified envelope may be processed, ACK'd, and acted upon with the confidence that its content was produced by the agent identified by `signature.signer`.

### 6.1 State Reasons Reference

| `state_reason` | Cause |
|----------------|-------|
| `signature_required` | Receiver is in `strict` mode; envelope was unsigned |
| `signer_from_mismatch` | `signature.signer` does not equal `from`; delegated signing is not supported in v1 |
| `key_not_found` | `signature.key_id` specified but no matching key in signer's capability document |
| `key_expired` | The selected public key's `expires_at` is in the past |
| `signature_verification_failed` | Ed25519 signature did not verify against the canonical body |

---

## 7. What Is NOT in Scope for v1

The following concerns are explicitly deferred. Implementations MUST NOT attempt to address them under this spec; doing so would create incompatible conventions that block a clean v2.

- **Key revocation.** There is no revocation list or revocation flag in v1. An expired `expires_at` is the only mechanism for retiring a key; deliberate revocation before expiry is out of scope.
- **Key rotation ceremony.** The process by which an agent transitions from one key pair to another — including overlap periods and notifying correspondents — is not defined. Agents may update their `keys` array; receivers will observe the change on the next capabilities fetch.
- **Certificate authorities and PKI.** There is no certificate chain, no CA signature, no trust anchor above the agent's own `capabilities.json`. Trust is rooted in capability discovery, not a PKI hierarchy.
- **Cross-runtime key discovery.** v1 assumes the receiver can fetch the signer's `capabilities.json` through the existing capability discovery path. Federated or cross-organization key discovery is deferred.
- **Delegated signing.** `signature.signer` MUST equal `from` in v1 and receivers MUST reject when they differ. Defining delegation semantics — where one agent signs on behalf of another — is deferred to a future extension.
- **Signature algorithm agility.** v1 mandates Ed25519 only. The `algorithm` field is included so future versions can introduce new algorithms without breaking the schema, but no migration or negotiation procedure is defined here.

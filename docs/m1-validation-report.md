# Agent Mail M1 Validation Report

Date: 2026-06-13
Channel: Discord `#agent-mail` (`1515485808650354878`)

## Summary

The M1 vertical slice is confirmed working across Clara/OpenClaw and KJB Agent:

```text
resolve -> send -> validate -> idempotency check -> store -> ACK -> reply -> ledger
```

## Validated Conversations

### Smoke Test 1

- Conversation: `conv_qpUpdeZ7ojx4E_le6pp27Q`
- Clara message: `msg_kgxjCX2KoZxEPX8WZLyhvw`
- KJB ACK: `msg_0yHvNYA58Qmev3jgBda76Q`
- Result: KJB validated required fields, body type/body consistency, sender allow-list, idempotency check, local store write, and ledger append. Clara validated and stored the ACK.

### Smoke Test 2

- Conversation: `conv_ypC81EuOVO-hpPkO03axQQ`
- Clara message: `msg_6Tfo3mV4MFUGAgHtVBlRBA`
- KJB ACK: `msg_Pa4BSNU6q5bibZYv8aLrgA`
- KJB substantive reply: `msg_mgL16_UVqH-tgA0LHD3lOA`
- Result: KJB validated and ACKed the inbound message. KJB then sent a substantive reply using the same `conversation_id` and `reply_to` pointing at the ACK. Clara validated and stored both.

## Idempotency

KJB Agent reran duplicate message delivery using already-seen `msg_` IDs and `conversation_id` values.

Result:

- duplicate messages were dropped
- no second ACK was sent
- no duplicate store write occurred
- conversations remained unchanged

Confirmed duplicate checks:

- `msg_kgxjCX2KoZxEPX8WZLyhvw`
- `msg_6Tfo3mV4MFUGAgHtVBlRBA`

## M1 Notes

- Both sides currently tolerate mutable `delivery_state` in message records.
- Before M2, split immutable envelope, mutable local message state, and append-only ledger events.
- Both sides agreed to use object-keyed address books: `{"mailboxes": {"mailbox-name": {...}}}`.

## M2 Candidate

M2 should expose the same operations over an MCP surface:

- `agent_mail.resolve`
- `agent_mail.send`
- `agent_mail.reply`
- `agent_mail.conversation.get`
- `agent_mail.conversation.list`
- `agent_mail.mailbox.capabilities`

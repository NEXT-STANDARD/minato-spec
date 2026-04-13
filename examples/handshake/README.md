# Example: Handshake

This example shows two Agent Cards exchanged during an `0x30 AGENT_HANDSHAKE`
(see [MINATO_PROTOCOL.md §4, §6](../../MINATO_PROTOCOL.md)).

## Scenario

Alice (Japanese, `default_trust_mode: suggest`) meets Bob (English, `default_trust_mode: plan`)
over BLE. Each device sends its Agent Card once; both sides persist the remote card so
subsequent messages in the session can skip re-handshake.

## Files

| File | Schema | Direction |
|------|--------|-----------|
| `alice-card.json` | [../../schema/agent-card.json](../../schema/agent-card.json) | A → B |
| `bob-card.json`   | [../../schema/agent-card.json](../../schema/agent-card.json) | B → A |

## Notes on values

- The `agent_id` / `signature` fields use realistic shapes (bech32 npub, 128-char hex Ed25519) but are **not** real keys — they are deterministic fakes derived from the strings "alice" and "bob". Do not copy them into production.
- `owner_locale` uses BCP 47. Bob's agent will translate outgoing content into Japanese when sending to Alice.
- Both cards advertise `language.translate`, which is a prerequisite for the multilingual flow in `../multilingual/`.

## Validation

```bash
npx -y ajv-cli@5 validate \
  -s ../../schema/agent-card.json \
  -r ../../schema/common.json \
  -d "alice-card.json"

npx -y ajv-cli@5 validate \
  -s ../../schema/agent-card.json \
  -r ../../schema/common.json \
  -d "bob-card.json"
```

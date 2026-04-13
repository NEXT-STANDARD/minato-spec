# Example: Multilingual message pair

Two `AGENT_MESSAGE` payloads that show the `content` / `translated_content`
convention from [MINATO_PROTOCOL.md §11](../../MINATO_PROTOCOL.md).

## Scenario

- Alice (`owner_locale: ja`) writes in Japanese. Her agent fills
  `translated_content` with the English rendering for Bob.
- Bob (`owner_locale: en`) replies in English. His agent fills
  `translated_content` with the Japanese rendering for Alice.

Neither user reads the other's language — the translation layer sits inside
the agent (typically the AI engine) and the protocol payload carries both
sides.

## Files

| File | Direction | `original_language` → receiver's `owner_locale` |
|------|-----------|------------------------------------------------|
| `ja-to-en-message.json` | Alice → Bob | `ja` → `en` |
| `en-to-ja-reply.json`   | Bob → Alice | `en` → `ja` |

Both validate against [../../schema/agent-message.json](../../schema/agent-message.json).

## Design notes

- The first message carries an optional `context.available_slots` array. `context`
  is intentionally free-form per §10 so agents can attach structured hints
  without minting a new intent.
- Receivers MUST prefer `translated_content` when it matches their
  `owner_locale`; otherwise they fall back to `content` (and may translate
  locally).
- Signatures cover the entire message, so altering a translation after
  signing invalidates the payload.

## Validation

```bash
for f in ja-to-en-message.json en-to-ja-reply.json; do
  npx -y ajv-cli@5 validate \
    -s ../../schema/agent-message.json \
    -r ../../schema/common.json \
    -d "$f"
done
```

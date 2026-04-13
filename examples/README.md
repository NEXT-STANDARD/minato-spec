# MINATO examples

Spec-compliant reference payloads you can paste into implementations,
fuzz against, or use as fixtures. Each subdirectory ships with a
`README.md` describing the scenario and the exact `ajv` command to
validate the files in that directory.

| Directory | Covers |
|-----------|--------|
| [`handshake/`](handshake/) | 0x30 AGENT_HANDSHAKE — two Agent Cards exchanged at session start. |
| [`schedule-negotiate/`](schedule-negotiate/) | 0x32 → 0x33 → 0x34 counter-proposal flow, tied together by one `request_id`. |
| [`multilingual/`](multilingual/) | 0x31 AGENT_MESSAGE pair showing the `content` / `translated_content` convention (§11). |

## Values to know

- All `agent_id` fields are **deterministic fakes** produced from short seeds
  ("alice", "bob"). They are valid bech32 shapes and validate against
  `schema/common.json`, but they correspond to no real Nostr keypair.
- All `signature` fields are **fake** but have the right shape (64 bytes,
  hex-encoded, 128 chars). Never copy them into production.
- Timestamps sit around April 2026 (`1712800000…`) to stay consistent across
  examples.

## Validating everything at once

From the repo root:

```bash
# handshake
npx -y ajv-cli@5 validate -s schema/agent-card.json -r schema/common.json \
  -d "examples/handshake/*-card.json"

# schedule negotiation
npx -y ajv-cli@5 validate -s schema/agent-request.json  -r schema/common.json -d examples/schedule-negotiate/01-request.json
npx -y ajv-cli@5 validate -s schema/agent-response.json -r schema/common.json -d examples/schedule-negotiate/02-response.json
npx -y ajv-cli@5 validate -s schema/agent-ack.json      -r schema/common.json -d examples/schedule-negotiate/03-ack.json

# multilingual
npx -y ajv-cli@5 validate -s schema/agent-message.json  -r schema/common.json \
  -d "examples/multilingual/*.json"
```

`ajv-cli@5` needs `ajv-formats` for the `date-time` format used inside
`proposed_event`. If your environment doesn't auto-install it, prepend:

```bash
npm i -g ajv-formats
```

or run through the Node API loading `ajv-formats` explicitly (see
`scripts/validate.mjs` if/when we add one).

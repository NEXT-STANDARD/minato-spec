# Example: Schedule Negotiation (counter-proposal flow)

A full three-message round demonstrating the `REQUEST → RESPONSE → ACK` pattern
described in [MINATO_PROTOCOL.md §6](../../MINATO_PROTOCOL.md) — specifically the
"Option B: Counter-proposal" branch.

## Scenario

Alice (`ja`) proposes drinks on Thursday 19:00 JST. Bob (`en`) counter-proposes
Friday 20:00 JST. Alice accepts.

All three messages share the same `request_id = "req_20260417_001"` so the
negotiation remains coherent even if the transport switches between BLE and
Nostr mid-flight.

## Files

| # | File | Type | Schema |
|---|------|------|--------|
| 1 | `01-request.json`  | `AGENT_REQUEST`  (0x32) | [../../schema/agent-request.json](../../schema/agent-request.json) |
| 2 | `02-response.json` | `AGENT_RESPONSE` (0x33) | [../../schema/agent-response.json](../../schema/agent-response.json) |
| 3 | `03-ack.json`      | `AGENT_ACK`      (0x34) | [../../schema/agent-ack.json](../../schema/agent-ack.json) |

## Trust Mode interaction

- Alice's outgoing request: her agent may draft the content automatically, but
  the `action: schedule.write` target makes this a **write-class** operation.
  In `plan` / `suggest`, Alice's owner approves the draft before it is sent.
- Bob's incoming request: same write-class rule applies. Bob's owner approves
  the counter-proposal before `02-response.json` is sent.
- Final ACK closes the negotiation on both sides. Only after ACK with
  `status: confirmed` should either agent write to its local calendar.

## Validation

```bash
for f in 01-request.json 02-response.json 03-ack.json; do
  name="${f#*-}"
  name="${name%.json}"
  npx -y ajv-cli@5 validate \
    -s "../../schema/agent-${name}.json" \
    -r ../../schema/common.json \
    -d "$f"
done
```

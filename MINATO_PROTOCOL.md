# MINATO Agent Protocol
## Specification v0.1

> **"A port where your agents meet the world."**  
> MINATO — Where Agents Meet

---

## Table of Contents

1. [Overview](#1-overview)
2. [Design Philosophy](#2-design-philosophy)
3. [Architecture](#3-architecture)
4. [Agent Card](#4-agent-card)
5. [Message Types](#5-message-types)
6. [Handshake Sequence](#6-handshake-sequence)
7. [Trust Mode](#7-trust-mode)
8. [Capabilities](#8-capabilities)
9. [Intents](#9-intents)
10. [Message Payload Specification](#10-message-payload-specification)
11. [Multilingual Support](#11-multilingual-support)
12. [Security](#12-security)
13. [Implementation Guidelines](#13-implementation-guidelines)
14. [Roadmap](#14-roadmap)

---

## 1. Overview

MINATO Agent Protocol is an open protocol that enables AI agents on different devices to communicate across platform and language boundaries.

```
Transport:      BLE Mesh (proximity) + Nostr (long-range)
Encryption:     Noise Protocol Framework (inherited from Bitchat)
Identity:       Nostr keypair (nsec / npub)
Data format:    JSON
Multilingual:   Protocol-level translation support
License:        MIT
```

### Core Principles

- **Decentralized** — No central server, no vendor lock-in
- **Open** — Any AI engine (Claude, GPT, Gemini, etc.) can connect
- **Privacy-first** — No account required; keypairs are identity
- **Language-agnostic** — Agents handle translation, removing human language barriers
- **Graduated trust** — Users control autonomy levels via Trust Mode

---

## 2. Design Philosophy

### Why MINATO?

"Minato" (港/湊) means "port" or "harbor" in Japanese — a place where people, goods, and information converge.  
MINATO aims to be the hub where agents meet: both a point of departure and arrival.

```
Personal app (now)
        ↓
Agent-to-agent communication platform (near future)
        ↓
Infrastructure connecting AI agents from any provider (vision)
```

### Historical Parallel

| Era    | What was standardized             |
|--------|-----------------------------------|
| 1970s  | TCP/IP (communication protocol)   |
| 1990s  | HTTP (common language of the Web) |
| 2000s  | REST API (service interconnection)|
| 2020s  | **MINATO (agent-to-agent)**       |

---

## 3. Architecture

```
┌─────────────────────────────────────────┐
│           Application Layer              │
│   SwiftUI (iOS) / Jetpack Compose (Android) │
├─────────────────────────────────────────┤
│             Agent Layer                  │
│   AgentCore / PermissionManager          │
│   CalendarAdapter / AIEngine             │
├──────────────────┬──────────────────────┤
│  MINATO Protocol │   Bitchat Protocol   │
│  (Agent-specific)│   (Chat, reused)     │
├──────────────────┴──────────────────────┤
│      BLE Mesh        │    Nostr Relay    │
│   (proximity, auto)  │ (long-range, auto)│
├─────────────────────────────────────────┤
│   Noise Protocol / Curve25519 / Ed25519  │
└─────────────────────────────────────────┘
```

### Transport Selection Logic

```
Is the peer physically nearby? (BLE detected)
        ↓ YES
    Use BLE Mesh (no internet required)
        ↓ NO
    Route via Nostr Relay (internet required)
```

---

## 4. Agent Card

An Agent Card is an agent's "business card."  
It is exchanged during the handshake and declares the agent's capabilities and trust settings.

### Schema

```json
{
  "minato_version": "0.1",
  "agent_id": "npub1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "display_name": "Phoenix's Agent",
  "owner_locale": "ja",
  "capabilities": [
    "schedule.read",
    "schedule.write",
    "message.reply",
    "language.translate"
  ],
  "default_trust_mode": "suggest",
  "supported_intents": [
    "schedule.negotiate",
    "schedule.confirm",
    "message.chat",
    "info.exchange"
  ],
  "ai_engine": "claude",
  "created_at": 1712800000,
  "ed25519_pub_key": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
  "signature": "ed25519_signature_here"
}
```

### Field Definitions

| Field                | Type     | Description                                    |
|----------------------|----------|------------------------------------------------|
| `minato_version`     | string   | Protocol version                               |
| `agent_id`           | string   | Nostr public key (npub format, secp256k1)      |
| `display_name`       | string   | User-configured agent name                     |
| `owner_locale`       | string   | Owner's primary language (BCP 47)              |
| `capabilities`       | string[] | List of permitted operations                   |
| `default_trust_mode` | string   | Default trust mode for new connections         |
| `supported_intents`  | string[] | List of supported intents                      |
| `ai_engine`          | string   | AI engine in use (informational)               |
| `created_at`         | int      | Unix timestamp                                 |
| `ed25519_pub_key`    | string   | Ed25519 public key (32 bytes, hex, 64 chars) used to verify `signature` |
| `signature`          | string   | Ed25519 signature over this card (anti-spoofing) |

> **Note on key separation**: `agent_id` is a Nostr/secp256k1 public key used as a stable identifier.
> `ed25519_pub_key` is the separate Ed25519 public key (CryptoKit `Curve25519.Signing`) used for
> cryptographic signature verification. Receivers MUST verify `signature` using `ed25519_pub_key`,
> not `agent_id`.

### Signature Canonical Form

The `signature` field is an Ed25519 signature over the **canonical serialization** of the Agent Card.
The canonical form is computed as follows:

1. Take the Agent Card JSON object.
2. Set `signature` to `null` (or omit it entirely).
3. Serialize to JSON with **sorted keys** (lexicographic, ascending) and no extra whitespace.
4. Sign the resulting UTF-8 bytes with the Ed25519 private key corresponding to `ed25519_pub_key`.
5. Encode the 64-byte signature as lowercase hex (128 characters).

Implementations MUST reject Agent Cards whose `signature` does not verify.

---

## 5. Message Types

The following message types are added on top of Bitchat's existing types.

| Code    | Name              | Description                              |
|---------|-------------------|------------------------------------------|
| `0x30`  | `AGENT_HANDSHAKE` | Initial connection and Agent Card exchange |
| `0x31`  | `AGENT_MESSAGE`   | General conversation and information sharing |
| `0x32`  | `AGENT_REQUEST`   | Action request (e.g., add a calendar event) |
| `0x33`  | `AGENT_RESPONSE`  | Response or proposal to a request        |
| `0x34`  | `AGENT_ACK`       | Confirmation (confirm) or rejection (reject) |
| `0x35`  | `AGENT_REVOKE`    | Permission revocation or disconnection   |
| `0x36`  | `AGENT_PING`      | Liveness check and latency measurement   |
| `0x37`  | `AGENT_LOG`       | Post-hoc notification (activity log in Full Auto mode) |

---

## 6. Handshake Sequence

### Proximity (BLE)

```
Device A (Phoenix)                   Device B (Friend)
        │                                  │
        │ ── 0x30 AGENT_HANDSHAKE ────────>│
        │    { Agent Card A }              │
        │                                  │
        │ <── 0x30 AGENT_HANDSHAKE ────────│
        │     { Agent Card B }             │
        │                                  │
        │  Both sides save Agent Cards     │
        │  Trust mode set to default       │
        │                                  │
        │ ── 0x31 AGENT_MESSAGE ──────────>│
        │    { intent: "message.chat" }    │
        │    "Hello!"                      │
```

### Long-Range (via Nostr)

```
Agent A                    Nostr Relay              Agent B
     │                             │                     │
     │ ── NIP-17 Encrypted DM ───>│ ── Deliver ────────>│
     │    { MINATO payload }       │                     │
     │                             │                     │
     │ <── NIP-17 Encrypted DM ───│ <── Deliver ────────│
     │     { MINATO payload }      │                     │
```

### Transport Selection and Fallback

All MINATO message types (0x30–0x37) are transport-independent. The same JSON payload flows over both BLE and Nostr.

```
Send decision:
  1. Check BLE reachability (isPeerConnected || isPeerReachable)
  2. If BLE reachable → send via BLE Mesh
  3. Else → send via Nostr Relay (NIP-17 gift-wrapped DM)

Receive:
  - BLE packets → BLEService handles directly
  - Nostr gift-wraps → decrypt NIP-17 → detect MINATO type (0x30–0x37) → dispatch
```

The `request_id` field ensures REQUEST→RESPONSE→ACK correlation across transport boundaries. A negotiation may start over BLE and complete over Nostr if the peer moves out of range.

### Schedule Negotiation Sequence

Multi-round negotiation for scheduling events (e.g., drinks, meetings):

```
Alice                                        Bob
  │                                            │
  │ ── 0x32 AGENT_REQUEST ───────────────────>│
  │    intent: "schedule.negotiate"            │
  │    action: "schedule.write"                │
  │    proposed_event: { Drinks, Thu 19:00 }   │
  │    request_id: "req_001"                   │
  │                                            │
  │         ┌─────────────────────────────┐    │
  │         │ Bob reviews proposal        │    │
  │         │ (Trust Mode gates approval) │    │
  │         └─────────────────────────────┘    │
  │                                            │
  ├── Option A: Accept ────────────────────────┤
  │ <── 0x34 AGENT_ACK ───────────────────────│
  │     request_id: "req_001"                  │
  │     status: "confirmed"                    │
  │                                            │
  ├── Option B: Counter-proposal ──────────────┤
  │ <── 0x33 AGENT_RESPONSE ──────────────────│
  │     request_id: "req_001"                  │
  │     proposed_event: { Drinks, Fri 20:00 }  │
  │                                            │
  │  Alice reviews counter-proposal            │
  │ ── 0x34 AGENT_ACK ───────────────────────>│
  │    request_id: "req_001"                   │
  │    status: "confirmed"                     │
  │                                            │
  ├── Option C: Decline ───────────────────────┤
  │ <── 0x34 AGENT_ACK ───────────────────────│
  │     request_id: "req_001"                  │
  │     status: "rejected"                     │
```

**Trust Mode interaction with schedule requests:**
- `plan` / `suggest`: Always require owner approval before responding
- `auto`: `schedule.write` and `schedule.delete` are high-risk — require approval even in auto mode
- `full_auto`: AI agent auto-responds, owner receives post-hoc AGENT_LOG

---

## 7. Trust Mode

Users can set a Trust Mode for each peer.  
The design is inspired by Claude Code's execution modes.

### Mode Overview

| Display Name | Technical Name | Auto-Execute | Confirmation Timing          |
|--------------|----------------|--------------|------------------------------|
| Apprentice   | `plan`         | No           | Confirm before every action  |
| Partner      | `suggest`      | No           | Confirm once before execution |
| Lieutenant   | `auto`         | Partial      | Confirm only for high-risk actions |
| Alter Ego    | `full_auto`    | Yes          | Post-hoc notification only   |

### Risk Assessment Criteria (for `auto` mode)

```
Auto-Execute (low risk)              Requires Confirmation (high risk)
───────────────────────              ─────────────────────────────────
schedule.read                        schedule.write
message.reply (known peer)           External service operations
info.exchange (public info)          Requests from new peers
                                     Sending personal information
```

### Settings Schema

```json
{
  "trust_settings": {
    "npub1bbb...": {
      "mode": "suggest",
      "custom_permissions": {
        "schedule.write": false
      },
      "established_at": 1712800000,
      "last_interaction": 1712900000
    }
  }
}
```

---

## 8. Capabilities

Capabilities are declared in the Agent Card and define what operations are permitted.  
Users manage these via ON/OFF toggles in the app settings.

### Calendar

| Capability        | Description                                   |
|-------------------|-----------------------------------------------|
| `schedule.read`   | Check availability (details not disclosed)    |
| `schedule.write`  | Add events to calendar                        |
| `schedule.delete` | Delete events (requires high trust mode)      |

### Messaging

| Capability         | Description                                  |
|--------------------|----------------------------------------------|
| `message.reply`    | Reply to the peer agent                      |
| `message.initiate` | Initiate conversation with the peer agent    |

### Information

| Capability          | Description                                  |
|---------------------|----------------------------------------------|
| `info.exchange`     | Exchange public info (name, language, etc.)   |
| `language.translate`| Automatic message translation                |
| `location.area`     | Share approximate location (city-level)       |
| `location.precise`  | Share precise location                        |

---

## 9. Intents

Intents are included in AGENT_MESSAGE / AGENT_REQUEST payloads to convey the purpose of a message.

| Intent                 | Description                              |
|------------------------|------------------------------------------|
| `message.chat`         | General conversation                     |
| `schedule.negotiate`   | Initiate schedule negotiation            |
| `schedule.confirm`     | Confirm a scheduled event                |
| `schedule.cancel`      | Cancel a scheduled event                 |
| `info.exchange`        | Information exchange / self-introduction |
| `trust.upgrade`        | Request Trust Mode upgrade               |
| `trust.downgrade`      | Downgrade Trust Mode                     |
| `connection.establish` | Establish a new connection               |
| `connection.terminate` | Terminate the connection                 |

---

## 10. Message Payload Specification

### AGENT_MESSAGE

```json
{
  "type": "AGENT_MESSAGE",
  "version": "0.1",
  "from": "npub1aaa...",
  "to": "npub1bbb...",
  "timestamp": 1712800000,
  "nonce": "random_nonce_for_replay_protection",
  "payload": {
    "intent": "schedule.negotiate",
    "content": "来週飲みに行きませんか？",
    "original_language": "ja",
    "translated_content": "Would you like to grab drinks next week?",
    "context": {
      "available_slots": [
        "2026-04-15T19:00:00+09:00",
        "2026-04-16T20:00:00+09:00"
      ]
    }
  },
  "signature": "ed25519_signature"
}
```

### AGENT_REQUEST

```json
{
  "type": "AGENT_REQUEST",
  "version": "0.1",
  "from": "npub1aaa...",
  "to": "npub1bbb...",
  "timestamp": 1712800000,
  "nonce": "random_nonce_for_replay_protection",
  "payload": {
    "request_id": "req_unique_id",
    "intent": "schedule.confirm",
    "action": "schedule.write",
    "content": "木曜19時で予定を入れてもいいですか？",
    "original_language": "ja",
    "translated_content": "Can I add a plan for Thursday at 7 PM?",
    "proposed_event": {
      "title": "Drinks",
      "start": "2026-04-17T19:00:00+09:00",
      "end": "2026-04-17T21:00:00+09:00",
      "location": "Shibuya"
    }
  },
  "signature": "ed25519_signature"
}
```

### AGENT_RESPONSE

Used for counter-proposals in schedule negotiation, or any response to an AGENT_REQUEST that isn't a simple confirm/reject.

```json
{
  "type": "AGENT_RESPONSE",
  "version": "0.1",
  "from": "npub1bbb...",
  "to": "npub1aaa...",
  "timestamp": 1712800050,
  "nonce": "random_nonce_for_replay_protection",
  "payload": {
    "request_id": "req_unique_id",
    "intent": "schedule.negotiate",
    "content": "木曜は難しいので金曜はどうですか？",
    "original_language": "ja",
    "translated_content": "Thursday is difficult — how about Friday?",
    "proposed_event": {
      "title": "Drinks",
      "start": "2026-04-18T20:00:00+09:00",
      "end": "2026-04-18T22:00:00+09:00",
      "location": "Shibuya"
    }
  },
  "signature": "ed25519_signature"
}
```

### AGENT_ACK

```json
{
  "type": "AGENT_ACK",
  "version": "0.1",
  "from": "npub1bbb...",
  "to": "npub1aaa...",
  "timestamp": 1712800100,
  "nonce": "random_nonce_for_replay_protection",
  "payload": {
    "request_id": "req_unique_id",
    "status": "confirmed",
    "content": "木曜19時、了解です！",
    "original_language": "ja",
    "translated_content": "Thursday at 7 PM, confirmed!"
  },
  "signature": "ed25519_signature"
}
```

### AGENT_REVOKE

Used to revoke previously granted trust, discard a remote Agent Card, or terminate the MINATO
agent relationship. Receivers MUST verify the envelope signature before applying a revoke.

```json
{
  "type": "AGENT_REVOKE",
  "version": "0.1",
  "from": "npub1aaa...",
  "to": "npub1bbb...",
  "timestamp": 1712800200,
  "nonce": "random_nonce_for_replay_protection",
  "payload": {
    "intent": "connection.terminate",
    "scope": "all",
    "reason": "Owner revoked this agent connection"
  },
  "signature": "ed25519_signature"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `intent` | string | MUST be `connection.terminate` |
| `scope` | string | Revocation scope: `trust`, `agent_card`, or `all` |
| `reason` | string | Optional human-readable reason shown in local logs/UI |

If `scope` is omitted, receivers SHOULD treat it as `all`.

### AGENT_LOG

Used for post-hoc notification of autonomous actions performed by an agent in `auto` or
`full_auto` trust mode. This is an audit signal, not a request for approval. Receivers MUST
verify the envelope signature before displaying or storing the log.

```json
{
  "type": "AGENT_LOG",
  "version": "0.1",
  "from": "npub1aaa...",
  "to": "npub1bbb...",
  "timestamp": 1712800300,
  "nonce": "random_nonce_for_replay_protection",
  "payload": {
    "log_id": "log_unique_id",
    "action": "auto_reply",
    "intent": "message.chat",
    "trust_mode": "full_auto",
    "content": "メッセージを受け取り、自動返信しました。",
    "context": {
      "source_message_id": "optional_message_id"
    }
  },
  "signature": "ed25519_signature"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `log_id` | string | Idempotency key for deduplicating repeated log delivery |
| `action` | string | Autonomous action type: `auto_reply`, `auto_schedule_ack`, or `auto_schedule_reject` |
| `intent` | string | Optional intent associated with the action |
| `trust_mode` | string | Trust mode under which the action was taken (`auto` or `full_auto`) |
| `content` | string | Human-readable summary of the action |
| `context` | object | Optional structured metadata |

Receivers SHOULD deduplicate logs by `(from, log_id)`.

---

## 11. Multilingual Support

MINATO eliminates language barriers at the protocol level.

### Basic Rules

1. `content` — Original text in the sender's native language
2. `original_language` — BCP 47 format (e.g., `ja`, `en`, `zh-TW`)
3. `translated_content` — Text translated into the receiver's language (optional)

### Translation Flow

```
Sending agent
  ↓
AI engine (e.g., Claude) generates translation
  ↓
Payload includes both original and translated content
  ↓
Receiving agent
  ↓
Displays translated_content matching receiver's language (owner_locale)
```

### Design Rationale

> With an agent as intermediary, language becomes just another data format.  
> Humans only need to speak their native tongue.

---

## 12. Security

### Encryption (inherited from Bitchat)

| Purpose             | Method                              |
|---------------------|-------------------------------------|
| Session establishment | Noise Protocol Framework (XX)     |
| Key exchange        | Curve25519                          |
| Message signing     | Ed25519                             |
| BLE encryption      | ChaCha20-Poly1305                   |
| Nostr DM            | NIP-17 (Sealed Gossip)              |
| Key storage         | iOS Keychain / Android Keystore     |

### Anti-Spoofing

- Ed25519 signature on every message
- Ed25519 signature on Agent Cards
- `nonce` for replay attack prevention
- `request_id` for deduplication

### Persistence

Agent Cards and Trust Settings MUST be persisted across app restarts.

| Data | Storage | Rationale |
|------|---------|-----------|
| Local Agent Card | iOS Keychain / Android Keystore | Survives reinstall, encrypted at rest |
| Remote Agent Cards | iOS Keychain / Android Keystore | Avoids re-handshake on every launch |
| Trust Settings (per npub) | iOS Keychain / Android Keystore | User's autonomy preferences are critical |
| Handshake state | In-memory only | Re-handshake on restart is acceptable |
| Pending replies | In-memory only | Transient approval queue, discard on restart |

On startup, the Agent Store loads persisted data. If the local Agent Card's `agent_id` no longer matches the device's current Nostr identity (e.g., after key rotation), a new card is created.

### Privacy Design

```
Granularity of information shared with peer agents (user-configurable):

Calendar: "available / busy" only        ← default
          Event titles
          Full details

Location: Not shared                     ← default
          City-level
          Precise location
```

---

## 13. Implementation Guidelines

### iOS Implementation

```
minato-ios/
  ├── bitchat/              ← Forked and reused as-is
  └── bitchat/MINATO/       ← Newly added
      ├── Models/
      │   ├── AgentCard.swift
      │   ├── Capability.swift
      │   ├── TrustMode.swift          (TrustMode enum + TrustSettings)
      │   └── Intent.swift
      ├── Services/
      │   ├── MINATOAgentStore.swift    (singleton: cards, trust, negotiations, persistence)
      │   └── BLEService+MINATO.swift   (packet routing + handlers + send methods)
      ├── AIEngine/
      │   ├── AIEngine.swift            (protocol: generateResponse, translate, extractSchedule)
      │   └── GeminiEngine.swift        (Gemini 2.5 Flash implementation)
      └── Protocol/
          └── MINATOMessageType.swift   (0x30–0x37 enum + MINATOPayload + PayloadContent)
```

### Android Implementation (Phase 2)

```
minato-android/
  ├── bitchat-android/  ← Forked and reused as-is
  └── minato/           ← Newly added
      ├── AgentCore.kt
      ├── AgentCard.kt
      ├── PermissionManager.kt
      ├── TrustModeManager.kt
      ├── CalendarAdapter.kt
      ├── engine/
      │   └── ClaudeEngine.kt
      └── protocol/
          ├── MINATOMessage.kt
          └── MINATOMessageType.kt
```

### What to Reuse from Bitchat

```
Reused (do not modify)
  ✅ All BLE communication (Core Bluetooth / Android BLE)
  ✅ Nostr protocol
  ✅ Noise Protocol encryption
  ✅ Store & Forward
  ✅ Message fragmentation
  ✅ Keypair management (Keychain / Keystore)

Newly added
  🆕 0x30–0x37 MINATO message types
  🆕 AgentCore (AI engine integration)
  🆕 Agent Card generation and exchange
  🆕 Trust Mode management
  🆕 Capabilities / Intent processing
  🆕 CalendarAdapter (EventKit / Android Calendar)
  🆕 Multilingual translation layer
  🆕 Approval UI (Apprentice through Alter Ego)
```

---

## 14. Roadmap

### Phase 0 — Preparation (1 week)
- [x] Fork Bitchat iOS and get it building
- [ ] Fork Bitchat Android and get it building
- [x] Understand and organize the codebase

### Phase 1 — BLE Agent Communication (2–3 weeks)
- [x] Implement MINATO message types (0x30–0x37)
- [x] Agent Card generation and exchange
- [x] Handshake sequence
- [x] Dummy agent with fixed responses (no API)
- [x] Verify the "connect with nearby agents" experience

### Phase 2 — Nostr Fallback (2 weeks)
- [x] Send/receive MINATO messages via Nostr
- [x] BLE / Nostr automatic switching
- [x] Trust Mode UI implementation

### Phase 3 — AI Integration (2 weeks)
- [x] AI engine integration (Gemini 2.5 Flash via AIEngine protocol)
- [x] AI-powered auto-reply with Trust Mode approval flow
- [x] Multilingual translation (protocol-level, AI engine handles)
- [x] Agent Card persistence (Keychain-backed)
- [x] Schedule negotiation flow (REQUEST → RESPONSE → ACK)
- [x] Schedule proposal UI (approval banner, counter-proposal sheet)
- [x] Nostr fallback for all MINATO message types (0x30–0x37)
- [ ] CalendarAdapter (EventKit read availability) — Phase 3.5
- [ ] AI schedule extraction from natural language — implemented, needs tuning

### Phase 4 — Polish & Platform (2 weeks)
- [ ] Agent Card QR code
- [ ] Activity log (for Full Auto mode)
- [ ] Android implementation catch-up

---

## Appendix A — Glossary

| Term              | Description                                                        |
|-------------------|--------------------------------------------------------------------|
| Agent Card        | An agent's business card. A JSON document declaring capabilities and trust settings |
| Trust Mode        | Autonomy level setting (Apprentice / Partner / Lieutenant / Alter Ego) |
| Capability        | A unit of operation permitted for an agent                         |
| Intent            | The purpose conveyed by a message (e.g., schedule.negotiate)       |
| BLE Mesh          | Proximity P2P communication via Bluetooth Low Energy               |
| Nostr             | Decentralized messaging protocol where keypairs are identity       |
| npub / nsec       | Nostr public key / secret key                                      |
| Handshake         | The Agent Card exchange sequence during initial connection         |
| Store & Forward   | Mechanism to cache messages while offline and deliver them later   |

---

## Appendix B — Reference Protocols & Projects

| Project                   | Referenced For                       |
|---------------------------|--------------------------------------|
| Bitchat (jackjackbits)    | BLE Mesh, Nostr, encryption layer    |
| bitchat-android           | Base for Android implementation      |
| Noise Protocol Framework  | Session establishment                |
| Nostr Protocol (NIPs)     | NIP-04, NIP-17 (encrypted DM)       |
| Google A2A Protocol       | Agent Card design reference          |
| Claude Code               | Trust Mode design reference          |

---

*MINATO Agent Protocol v0.1*  
*Status: Draft*  
*Last updated: 2026-04-12*  
*Japanese version: [docs/ja/MINATO_PROTOCOL.md](docs/ja/MINATO_PROTOCOL.md)*

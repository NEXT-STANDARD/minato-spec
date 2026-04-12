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
  "signature": "ed25519_signature_here"
}
```

### Field Definitions

| Field                | Type     | Description                                    |
|----------------------|----------|------------------------------------------------|
| `minato_version`     | string   | Protocol version                               |
| `agent_id`           | string   | Nostr public key (npub format)                 |
| `display_name`       | string   | User-configured agent name                     |
| `owner_locale`       | string   | Owner's primary language (BCP 47)              |
| `capabilities`       | string[] | List of permitted operations                   |
| `default_trust_mode` | string   | Default trust mode for new connections         |
| `supported_intents`  | string[] | List of supported intents                      |
| `ai_engine`          | string   | AI engine in use (informational)               |
| `created_at`         | int      | Unix timestamp                                 |
| `signature`          | string   | Ed25519 signature (anti-spoofing)              |

---

## 5. Message Types

The following message types are added on top of Bitchat's existing types.

| Code    | Name              | Description                              |
|---------|-------------------|------------------------------------------|
| `0x10`  | `AGENT_HANDSHAKE` | Initial connection and Agent Card exchange |
| `0x11`  | `AGENT_MESSAGE`   | General conversation and information sharing |
| `0x12`  | `AGENT_REQUEST`   | Action request (e.g., add a calendar event) |
| `0x13`  | `AGENT_RESPONSE`  | Response or proposal to a request        |
| `0x14`  | `AGENT_ACK`       | Confirmation (confirm) or rejection (reject) |
| `0x15`  | `AGENT_REVOKE`    | Permission revocation or disconnection   |
| `0x16`  | `AGENT_PING`      | Liveness check and latency measurement   |
| `0x17`  | `AGENT_LOG`       | Post-hoc notification (activity log in Full Auto mode) |

---

## 6. Handshake Sequence

### Proximity (BLE)

```
Device A (Phoenix)                   Device B (Friend)
        │                                  │
        │ ── 0x10 AGENT_HANDSHAKE ────────>│
        │    { Agent Card A }              │
        │                                  │
        │ <── 0x10 AGENT_HANDSHAKE ────────│
        │     { Agent Card B }             │
        │                                  │
        │  Both sides save Agent Cards     │
        │  Trust mode set to default       │
        │                                  │
        │ ── 0x11 AGENT_MESSAGE ──────────>│
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
  "request_id": "req_unique_id",
  "payload": {
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

### AGENT_ACK

```json
{
  "type": "AGENT_ACK",
  "version": "0.1",
  "from": "npub1bbb...",
  "to": "npub1aaa...",
  "timestamp": 1712800100,
  "request_id": "req_unique_id",
  "payload": {
    "status": "confirmed",
    "content": "木曜19時、了解です！",
    "original_language": "ja",
    "translated_content": "Thursday at 7 PM, confirmed!"
  },
  "signature": "ed25519_signature"
}
```

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

### iOS Implementation (Phase 1)

```
minato-ios/
  ├── Bitchat/          ← Forked and reused as-is
  └── MINATO/           ← Newly added
      ├── AgentCore.swift
      ├── AgentCard.swift
      ├── PermissionManager.swift
      ├── TrustModeManager.swift
      ├── CalendarAdapter.swift
      ├── AIEngine/
      │   └── ClaudeEngine.swift
      └── Protocol/
          ├── MINATOMessage.swift
          └── MINATOMessageType.swift
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
  🆕 0x10–0x17 MINATO message types
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
- [ ] Understand and organize the codebase

### Phase 1 — BLE Agent Communication (2–3 weeks)
- [ ] Implement MINATO message types (0x10–0x15)
- [ ] Agent Card generation and exchange
- [ ] Handshake sequence
- [ ] Dummy agent with fixed responses (no API)
- [ ] Verify the "connect with nearby agents" experience

### Phase 2 — Nostr Fallback (2 weeks)
- [ ] Send/receive MINATO messages via Nostr
- [ ] BLE / Nostr automatic switching
- [ ] Trust Mode UI implementation

### Phase 3 — AI Integration (2 weeks)
- [ ] Claude API connection (AgentCore)
- [ ] CalendarAdapter (read availability)
- [ ] Schedule negotiation flow (negotiate → confirm)

### Phase 4 — Multilingual & Polish (2 weeks)
- [ ] Multilingual translation layer
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

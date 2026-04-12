# MINATO Agent Protocol

> **Open protocol for agent-to-agent communication across devices.**  
> 湊 — Where Agents Meet.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Status: Draft](https://img.shields.io/badge/Status-Draft-yellow.svg)]()
[![Version: 0.1](https://img.shields.io/badge/Version-0.1-green.svg)]()

---

## What is MINATO?

MINATO is an open protocol that enables AI agents on different devices to communicate across platform and **language barriers**.

```
Your Phone                      Friend's Phone
┌─────────────┐                ┌─────────────┐
│             │                │             │
│  Your       │◄──MINATO──────►│  Friend's   │
│  Agent      │                │  Agent      │
│             │  BLE / Nostr   │             │
└─────────────┘                └─────────────┘
       │                              │
  Speaks Japanese               Speaks English
       │                              │
       └──── Language barrier: GONE ──┘
```

With an agent as intermediary, humans only need to speak their native tongue.

---

## Key Features

- **📡 Dual Transport** — BLE Mesh (proximity) + Nostr (long-range) with automatic switching
- **🔐 Privacy First** — No account required; Nostr keypairs are identity
- **🌍 Language Agnostic** — Protocol-level multilingual translation support
- **🤝 Trust Modes** — Apprentice / Partner / Lieutenant / Alter Ego (graduated autonomy control)
- **🔓 Open** — Any AI engine (Claude, GPT, Gemini, etc.) can connect
- **📱 Cross-Platform** — iOS, Android, Desktop

---

## Trust Modes

Users can set an autonomy level for each peer's agent.

| Display Name | Technical Name | Behavior |
|--------------|----------------|----------|
| Apprentice   | `plan`         | Suggests only. You decide everything |
| Partner      | `suggest`      | Asks for confirmation once before executing |
| Lieutenant   | `auto`         | Handles small tasks automatically, asks for important ones |
| Alter Ego    | `full_auto`    | Fully autonomous. Post-hoc notification only |

---

## Architecture

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
│  (Agent-specific)│   (Transport base)   │
├──────────────────┴──────────────────────┤
│      BLE Mesh        │    Nostr Relay    │
├─────────────────────────────────────────┤
│   Noise Protocol / Curve25519 / Ed25519  │
└─────────────────────────────────────────┘
```

---

## Specification

👉 **[MINATO_PROTOCOL.md](MINATO_PROTOCOL.md)** — Full protocol specification (English)

📖 **[日本語版](docs/ja/MINATO_PROTOCOL.md)** — Japanese version

---

## Implementations

| Platform | Repository | Language | Status |
|----------|------------|----------|--------|
| iOS | [minato-ios](https://github.com/NEXT-STANDARD/minato-ios) | Swift | 🚧 Phase 0 Complete |
| Android | [minato-android](https://github.com/NEXT-STANDARD/minato-android) | Kotlin | 🚧 Coming Soon |

---

## Message Types

| Code | Name | Description |
|------|------|-------------|
| `0x10` | `AGENT_HANDSHAKE` | Initial connection and Agent Card exchange |
| `0x11` | `AGENT_MESSAGE` | General conversation and information sharing |
| `0x12` | `AGENT_REQUEST` | Action request (e.g., add calendar event) |
| `0x13` | `AGENT_RESPONSE` | Response or proposal to a request |
| `0x14` | `AGENT_ACK` | Confirmation or rejection |
| `0x15` | `AGENT_REVOKE` | Permission revocation or disconnection |
| `0x16` | `AGENT_PING` | Liveness check |
| `0x17` | `AGENT_LOG` | Post-hoc notification (Full Auto mode) |

---

## Vision

> "Minato" (港) means "port" in Japanese — a place where people, goods, and information converge.  
> MINATO is the hub where agents meet: both a point of departure and arrival.

| Era | What was standardized |
|-----|----------------------|
| 1970s | TCP/IP |
| 1990s | HTTP |
| 2000s | REST API |
| **2020s** | **MINATO — Agent-to-Agent** |

---

## Contributing

MINATO is an open protocol.  
Propose ideas and discuss in [Issues](https://github.com/NEXT-STANDARD/minato-spec/issues). Implementation PRs are welcome.

1. Fork this repository
2. Create your feature branch (`git checkout -b feature/your-idea`)
3. Commit your changes
4. Open a Pull Request

---

## Built On

- [Bitchat](https://github.com/jackjackbits/bitchat) — BLE Mesh transport
- [bitchat-android](https://github.com/permissionlesstech/bitchat-android) — Android transport
- [Nostr Protocol](https://github.com/nostr-protocol/nostr) — Decentralized relay
- [Noise Protocol Framework](https://noiseprotocol.org/) — E2E encryption

---

## License

MIT License — see [LICENSE](LICENSE)

---

*MINATO Agent Protocol v0.1 — Draft*  
*by [NEXT-STANDARD](https://github.com/NEXT-STANDARD)*

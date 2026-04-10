# MINATO Agent Protocol

> **Open protocol for agent-to-agent communication across devices.**  
> 湊 — Where Agents Meet.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Status: Draft](https://img.shields.io/badge/Status-Draft-yellow.svg)]()
[![Version: 0.1](https://img.shields.io/badge/Version-0.1-green.svg)]()

---

## What is MINATO?

MINATOは、異なるデバイス上のAIエージェント同士が、  
プラットフォームや**言語の壁を超えて**通信するためのオープンプロトコルです。

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

エージェントが間に入ることで、人間は母語で話すだけでいい。

---

## Key Features

- **📡 Dual Transport** — BLE Mesh（近距離）+ Nostr（遠距離）を自動切替
- **🔐 Privacy First** — アカウント不要、Nostr鍵ペアがアイデンティティ
- **🌍 Language Agnostic** — プロトコルレベルで多言語翻訳をサポート
- **🤝 Trust Modes** — 見習い / 相棒 / 右腕 / 分身（自律度を段階的に制御）
- **🔓 Open** — Claude / GPT / Gemini など任意のAIエンジンが接続可能
- **📱 Cross-Platform** — iOS, Android, Desktop 対応

---

## Trust Modes

ユーザーは相手ごとにエージェントの自律度を設定できます。

| 表示名 | 技術名 | 動作 |
|--------|--------|------|
| 見習い | `plan` | 提案するだけ。実行は全部あなたが決める |
| 相棒 | `suggest` | アクション直前に一度だけ確認する |
| 右腕 | `auto` | 小さなことは自動、大事なことだけ聞く |
| 分身 | `full_auto` | 全部任せる。事後通知のみ |

---

## Architecture

```
┌─────────────────────────────────────────┐
│           Application Layer              │
│   SwiftUI（iOS）/ Jetpack Compose（Android）│
├─────────────────────────────────────────┤
│             Agent Layer                  │
│   AgentCore / PermissionManager          │
│   CalendarAdapter / AIEngine             │
├──────────────────┬──────────────────────┤
│  MINATO Protocol │   Bitchat Protocol   │
│  （Agent専用）   │   （Transport基盤）  │
├──────────────────┴──────────────────────┤
│      BLE Mesh        │    Nostr Relay    │
├─────────────────────────────────────────┤
│   Noise Protocol / Curve25519 / Ed25519  │
└─────────────────────────────────────────┘
```

---

## Specification

👉 **[MINATO_PROTOCOL.md](MINATO_PROTOCOL.md)** — Full protocol specification

---

## Implementations

| Platform | Repository | Language | Status |
|----------|------------|----------|--------|
| iOS | [minato-ios](https://github.com/NEXT-STANDARD/minato-ios) | Swift | 🚧 Coming Soon |
| Android | [minato-android](https://github.com/NEXT-STANDARD/minato-android) | Kotlin | 🚧 Coming Soon |

---

## Message Types

| Code | Name | Description |
|------|------|-------------|
| `0x10` | `AGENT_HANDSHAKE` | 初回接続・Agent Card交換 |
| `0x11` | `AGENT_MESSAGE` | 通常の会話・情報共有 |
| `0x12` | `AGENT_REQUEST` | アクション要求 |
| `0x13` | `AGENT_RESPONSE` | 要求への応答 |
| `0x14` | `AGENT_ACK` | 承認 / 拒否 |
| `0x15` | `AGENT_REVOKE` | 権限取り消し・切断 |
| `0x16` | `AGENT_PING` | 死活確認 |
| `0x17` | `AGENT_LOG` | 事後通知（Full Auto時）|

---

## Vision

> 港（みなと）は、人・物・金・情報が集まる場所。  
> MINATOは、エージェントが集まるハブ。  
> 出発点であり、到着点でもあるインフラを目指します。

| Era | What was standardized |
|-----|----------------------|
| 1970s | TCP/IP |
| 1990s | HTTP |
| 2000s | REST API |
| **2020s** | **MINATO — Agent-to-Agent** |

---

## Contributing

MINATOはオープンプロトコルです。  
仕様への提案・議論はIssuesへ。実装のPRも歓迎します。

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

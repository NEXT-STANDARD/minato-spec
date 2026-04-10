# MINATO Agent Protocol
## 仕様書 v0.1

> **「あなたのエージェントが、世界と繋がる港」**  
> MINATO — Where Agents Meet

---

## 目次

1. [Overview](#1-overview)
2. [設計思想](#2-設計思想)
3. [アーキテクチャ](#3-アーキテクチャ)
4. [Agent Card](#4-agent-card)
5. [メッセージタイプ](#5-メッセージタイプ)
6. [ハンドシェイクシーケンス](#6-ハンドシェイクシーケンス)
7. [Trust Mode（信頼モード）](#7-trust-mode信頼モード)
8. [Capabilities（権限定義）](#8-capabilities権限定義)
9. [Intent（意図定義）](#9-intent意図定義)
10. [メッセージPayload仕様](#10-メッセージpayload仕様)
11. [多言語対応](#11-多言語対応)
12. [セキュリティ](#12-セキュリティ)
13. [実装ガイドライン](#13-実装ガイドライン)
14. [ロードマップ](#14-ロードマップ)

---

## 1. Overview

MINATO Agent Protocolは、異なるデバイス上のAIエージェント同士が、プラットフォームや言語の壁を超えて通信するためのオープンプロトコルです。

```
Transport:      BLE Mesh（近距離）+ Nostr（遠距離）
暗号化:         Noise Protocol Framework（Bitchat継承）
アイデンティティ: Nostr keypair（nsec / npub）
データ形式:     JSON
言語対応:       プロトコルレベルで多言語翻訳をサポート
ライセンス:     MIT
```

### 基本原則

- **分散型** — 中央サーバー不要、特定企業に依存しない
- **オープン** — Claude / GPT / Gemini など任意のAIエンジンが接続可能
- **プライバシーファースト** — アカウント不要、鍵ペアがアイデンティティ
- **言語非依存** — エージェントが翻訳を担うことで人間の言語の壁を消す
- **段階的な信頼** — Trust Modeによってユーザーが自律度を制御できる

---

## 2. 設計思想

### なぜMINATOか

「湊（みなと）」は人・物・金・情報が集まる場所です。  
MINATOはエージェントが集まるハブ、出発点であり到着点でもあるインフラを目指します。

```
個人向けアプリ（今）
        ↓
エージェント間通信プラットフォーム（近い将来）
        ↓
各社AIエージェントが接続するインフラ（理想）
```

### インターネットの歴史との対比

| 時代   | 標準化されたもの        |
|--------|-------------------------|
| 1970s  | TCP/IP（通信プロトコル）|
| 1990s  | HTTP（Webの共通言語）   |
| 2000s  | REST API（サービス間接続）|
| 2020s  | **MINATO（エージェント間）** |

---

## 3. アーキテクチャ

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
│  （Agent専用）   │   （Chat流用）       │
├──────────────────┴──────────────────────┤
│      BLE Mesh        │    Nostr Relay    │
│    （近距離・自動）  │  （遠距離・自動） │
├─────────────────────────────────────────┤
│   Noise Protocol / Curve25519 / Ed25519  │
└─────────────────────────────────────────┘
```

### Transport選択ロジック

```
相手が物理的に近い（BLE検出）
        ↓ YES
    BLE Mesh使用（インターネット不要）
        ↓ NO
    Nostr Relay経由（インターネット使用）
```

---

## 4. Agent Card

Agent Cardはエージェントの「名刺」です。  
ハンドシェイク時に交換され、相手エージェントの能力・信頼設定を宣言します。

### スキーマ

```json
{
  "minato_version": "0.1",
  "agent_id": "npub1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "display_name": "フェニックスのエージェント",
  "owner_locale": "ja",
  "capabilities": [
    "schedule.read",
    "schedule.write",
    "message.reply",
    "language.translate"
  ],
  "default_trust_mode": "aibou",
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

### フィールド定義

| フィールド           | 型       | 説明                                      |
|----------------------|----------|-------------------------------------------|
| `minato_version`     | string   | プロトコルバージョン                      |
| `agent_id`           | string   | Nostr公開鍵（npub形式）                   |
| `display_name`       | string   | ユーザーが設定したエージェント名          |
| `owner_locale`       | string   | オーナーの主要言語（BCP 47）              |
| `capabilities`       | string[] | 許可する操作の一覧                        |
| `default_trust_mode` | string   | デフォルトの信頼モード                    |
| `supported_intents`  | string[] | 対応するIntentの一覧                      |
| `ai_engine`          | string   | 使用しているAIエンジン（参考情報）        |
| `created_at`         | int      | Unix timestamp                            |
| `signature`          | string   | Ed25519による署名（なりすまし防止）       |

---

## 5. メッセージタイプ

Bitchatの既存メッセージタイプに以下を追加します。

| コード  | 名前              | 説明                               |
|---------|-------------------|------------------------------------|
| `0x10`  | `AGENT_HANDSHAKE` | 初回接続・Agent Card交換           |
| `0x11`  | `AGENT_MESSAGE`   | 通常の会話・情報共有               |
| `0x12`  | `AGENT_REQUEST`   | アクション要求（予定追加など）     |
| `0x13`  | `AGENT_RESPONSE`  | 要求への応答・提案                 |
| `0x14`  | `AGENT_ACK`       | 承認（confirm）または拒否（reject）|
| `0x15`  | `AGENT_REVOKE`    | 権限取り消し・接続切断             |
| `0x16`  | `AGENT_PING`      | 死活確認・レイテンシ計測           |
| `0x17`  | `AGENT_LOG`       | 事後通知（Full Auto時の活動記録）  |

---

## 6. ハンドシェイクシーケンス

### 近距離（BLE）

```
デバイスA（フェニックス）          デバイスB（友人）
        │                                  │
        │ ── 0x10 AGENT_HANDSHAKE ────────>│
        │    { Agent Card A }              │
        │                                  │
        │ <── 0x10 AGENT_HANDSHAKE ────────│
        │     { Agent Card B }             │
        │                                  │
        │  双方がAgent Cardを保存          │
        │  trust_modeをデフォルト設定      │
        │                                  │
        │ ── 0x11 AGENT_MESSAGE ──────────>│
        │    { intent: "message.chat" }    │
        │    「こんにちは！」              │
```

### 遠距離（Nostr経由）

```
エージェントA                  Nostrリレー              エージェントB
     │                             │                         │
     │ ── NIP-17 暗号化DM ────────>│ ── 配送 ──────────────>│
     │    { MINATO payload }       │                         │
     │                             │                         │
     │ <── NIP-17 暗号化DM ────────│ <── 配送 ───────────────│
     │     { MINATO payload }      │                         │
```

---

## 7. Trust Mode（信頼モード）

ユーザーは相手ごとにTrust Modeを設定できます。  
Claude Codeの実行モードを参考に設計しています。

### モード一覧

| 表示名（JP） | 技術名      | 自動実行 | 確認タイミング           |
|--------------|-------------|----------|--------------------------|
| 見習い       | `plan`      | ❌       | 全アクション前に確認     |
| 相棒         | `suggest`   | ❌       | 実行直前に1回確認        |
| 右腕         | `auto`      | ✅（一部）| 高リスク操作のみ確認    |
| 分身         | `full_auto` | ✅       | 事後通知のみ             |

### リスク判定基準（右腕モード用）

```
自動実行OK（低リスク）          要確認（高リスク）
─────────────────────          ────────────────
schedule.read                  schedule.write
message.reply（既知の相手）    外部サービス操作
info.exchange（公開情報）      新しい相手からのリクエスト
                               個人情報の送信
```

### 設定スキーマ

```json
{
  "trust_settings": {
    "npub1bbb...": {
      "mode": "aibou",
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

## 8. Capabilities（権限定義）

Agent Cardで宣言する権限の一覧です。  
ユーザーはアプリの設定画面でON/OFFを管理できます。

### カレンダー系

| Capability        | 説明                               |
|-------------------|------------------------------------|
| `schedule.read`   | 空き時間の確認（詳細は非公開）     |
| `schedule.write`  | カレンダーへの予定追加             |
| `schedule.delete` | 予定の削除（要：高い信頼モード）   |

### メッセージ系

| Capability         | 説明                               |
|--------------------|------------------------------------|
| `message.reply`    | 相手エージェントへの返答           |
| `message.initiate` | こちらからエージェントに話しかける |

### 情報系

| Capability          | 説明                               |
|---------------------|------------------------------------|
| `info.exchange`     | 公開情報の交換（名前・言語など）   |
| `language.translate`| メッセージの自動翻訳               |
| `location.area`     | 大まかな地域の共有（市区町村レベル）|
| `location.precise`  | 詳細な位置情報の共有               |

---

## 9. Intent（意図定義）

AGENT_MESSAGE / AGENT_REQUESTのpayloadに含まれる意図の一覧です。

| Intent                 | 説明                               |
|------------------------|------------------------------------|
| `message.chat`         | 通常の会話                         |
| `schedule.negotiate`   | 日程調整の開始                     |
| `schedule.confirm`     | 日程の確定                         |
| `schedule.cancel`      | 予定のキャンセル                   |
| `info.exchange`        | 情報交換・自己紹介                 |
| `trust.upgrade`        | Trust Modeの昇格リクエスト         |
| `trust.downgrade`      | Trust Modeの降格                   |
| `connection.establish` | 新規接続の確立                     |
| `connection.terminate` | 接続の終了                         |

---

## 10. メッセージPayload仕様

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
      "title": "飲み会",
      "start": "2026-04-17T19:00:00+09:00",
      "end": "2026-04-17T21:00:00+09:00",
      "location": "渋谷"
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

## 11. 多言語対応

MINATOはプロトコルレベルで言語の壁を解消します。

### 基本ルール

1. `content` — 送信者の母語で書かれたオリジナル
2. `original_language` — BCP 47形式（例：`ja`, `en`, `zh-TW`）
3. `translated_content` — 受信者の言語に翻訳済みのテキスト（任意）

### 翻訳フロー

```
送信エージェント
  ↓
AIエンジン（Claude等）で翻訳を生成
  ↓
payload に original + translated を両方含めて送信
  ↓
受信エージェント
  ↓
受信者の言語（owner_locale）のtranslated_contentを表示
```

### 設計の意義

> エージェントが間に入ることで、言語はただのデータ形式になる。  
> 人間は母語で話すだけでいい。

---

## 12. セキュリティ

### 暗号化（Bitchat継承）

| 用途           | 方式                              |
|----------------|-----------------------------------|
| セッション確立 | Noise Protocol Framework（XX）    |
| 鍵交換         | Curve25519                        |
| メッセージ署名 | Ed25519                           |
| BLE暗号化      | ChaCha20-Poly1305                 |
| Nostr DM       | NIP-17（Sealed Gossip）           |
| 鍵保管         | iOS Keychain / Android Keystore   |

### なりすまし防止

- 全メッセージにEd25519署名を付与
- Agent CardにもEd25519署名を付与
- `nonce`によるリプレイアタック防止
- `request_id`による重複処理防止

### プライバシー設計

```
相手エージェントに渡す情報の粒度（ユーザーが設定）

カレンダー: 「空き / 埋まり」のみ  ←  デフォルト
            予定のタイトルまで
            詳細情報まで

位置情報:   共有しない             ←  デフォルト
            市区町村レベル
            詳細位置
```

---

## 13. 実装ガイドライン

### iOS実装（Phase 1）

```
minato-ios/
  ├── Bitchat/          ← forkしてそのまま流用
  └── MINATO/           ← 新規追加
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

### Android実装（Phase 2）

```
minato-android/
  ├── bitchat-android/  ← forkしてそのまま流用
  └── minato/           ← 新規追加
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

### Bitchatからの流用範囲

```
流用（変更しない）
  ✅ BLE通信全般（Core Bluetooth / Android BLE）
  ✅ Nostrプロトコル
  ✅ Noise Protocol暗号化
  ✅ Store & Forward
  ✅ メッセージフラグメント処理
  ✅ 鍵ペア管理（Keychain / Keystore）

新規追加
  🆕 0x10〜0x17 MINATOメッセージタイプ
  🆕 AgentCore（AIエンジン連携）
  🆕 Agent Card生成・交換
  🆕 Trust Mode管理
  🆕 Capabilities / Intent処理
  🆕 CalendarAdapter（EventKit / Android Calendar）
  🆕 多言語翻訳レイヤー
  🆕 承認UI（見習い〜分身）
```

---

## 14. ロードマップ

### Phase 0 — 準備（1週間）
- [ ] Bitchat iOS forkしてビルドを通す
- [ ] Bitchat Android forkしてビルドを通す
- [ ] コードベースの理解・整理

### Phase 1 — BLEエージェント通信（2〜3週間）
- [ ] MINATOメッセージタイプの実装（0x10〜0x15）
- [ ] Agent Card生成・交換
- [ ] ハンドシェイクシーケンス
- [ ] 固定応答のダミーエージェント（APIなし）
- [ ] 「近くにいる人のエージェントと繋がる」体験確認

### Phase 2 — Nostrフォールバック（2週間）
- [ ] Nostr経由のMINATOメッセージ送受信
- [ ] BLE / Nostr自動切替
- [ ] Trust Mode UIの実装

### Phase 3 — AI連携（2週間）
- [ ] Claude API接続（AgentCore）
- [ ] CalendarAdapter（空き時間の読み取り）
- [ ] スケジュール交渉フロー（negotiate → confirm）

### Phase 4 — 多言語・仕上げ（2週間）
- [ ] 多言語翻訳レイヤー
- [ ] Agent Card QRコード
- [ ] 事後ログ（Full Auto時）
- [ ] Android版の追いかけ実装

---

## Appendix A — 用語集

| 用語              | 説明                                                      |
|-------------------|-----------------------------------------------------------|
| Agent Card        | エージェントの名刺。能力・信頼設定を宣言するJSON          |
| Trust Mode        | エージェントの自律度設定（見習い / 相棒 / 右腕 / 分身）   |
| Capability        | エージェントに許可する操作の単位                          |
| Intent            | メッセージが表す意図（schedule.negotiateなど）            |
| BLE Mesh          | Bluetooth Low Energyによる近距離P2P通信                   |
| Nostr             | 分散型メッセージプロトコル。鍵ペアがアイデンティティ      |
| npub / nsec       | Nostrの公開鍵 / 秘密鍵                                    |
| Handshake         | 初回接続時のAgent Card交換シーケンス                      |
| Store & Forward   | オフライン時にメッセージをキャッシュして後で届ける仕組み  |

---

## Appendix B — 参考プロトコル・プロジェクト

| プロジェクト              | 参照箇所                        |
|---------------------------|---------------------------------|
| Bitchat (jackjackbits)    | BLE Mesh, Nostr, 暗号化全般     |
| bitchat-android           | Android実装のベース             |
| Noise Protocol Framework  | セッション確立                  |
| Nostr Protocol (NIPs)     | NIP-04, NIP-17（暗号化DM）      |
| Google A2A Protocol       | Agent Card設計の参考            |
| Claude Code               | Trust Mode設計の参考            |

---

*MINATO Agent Protocol v0.1*  
*ステータス: Draft*  
*最終更新: 2026-04-11*

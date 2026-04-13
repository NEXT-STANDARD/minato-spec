# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## このリポジトリについて

`minato-spec` は **MINATO Agent Protocol の仕様リポジトリ** です。
ソースコードは含まれず、仕様書・JSON Schema・サンプルペイロード・ガバナンス文書のみを管理します。
実装は別リポジトリ（`minato-ios`, `minato-android`）で行われる予定です。

仕様本体は `MINATO_PROTOCOL.md` が単一の正典（source of truth）です。
変更する際は必ず `MINATO_PROTOCOL.md` を最初に更新し、派生ファイル（スキーマ、サンプル、README）をそれに追従させてください。

## ビルド / テスト

このリポジトリにはビルド・リントのコマンドはありません（Markdown と JSON のみ）。
JSON Schema とサンプルの妥当性は `ajv` で検証できます:

```bash
# ajv 8 + ajv-formats（date-time format を使うので必須）
npm i -g ajv-cli ajv-formats

# 例: handshake の Agent Card を検証
npx ajv-cli validate -c ajv-formats \
  -s schema/agent-card.json -r schema/common.json \
  -d "examples/handshake/*-card.json"
```

`schema/common.json` が共通定義（npub, trust_mode, capability, intent,
proposed_event など）を持ち、他のスキーマは `"$ref": "common.json#/..."`
で参照しているので、検証時には必ず `-r schema/common.json` を付けてください。

## 現在のディレクトリ状態

```
minato-spec/
├── CLAUDE.md               ← このファイル
├── LICENSE                 ← MIT
├── MINATO_PROTOCOL.md      ← 仕様本体（v0.1 Draft）
├── README.md               ← プロジェクト紹介
├── docs/
│   ├── ja/                 ← 日本語の人間向け解説
│   └── en/                 ← 英語の人間向け解説
├── schema/
│   ├── common.json         ← 共有定義（npub, trust_mode, capability, intent 他）
│   ├── agent-card.json     ← Agent Card (§4)
│   ├── agent-message.json  ← AGENT_MESSAGE (0x31)
│   ├── agent-request.json  ← AGENT_REQUEST (0x32)
│   ├── agent-response.json ← AGENT_RESPONSE (0x33)
│   └── agent-ack.json      ← AGENT_ACK (0x34)
└── examples/
    ├── handshake/          ← Agent Card ペア交換
    ├── schedule-negotiate/ ← REQUEST → RESPONSE → ACK (同一 request_id)
    └── multilingual/       ← content / translated_content の対
```

以下は README の Roadmap で予定されているが **未作成** です:

```
schema/agent-handshake.json     ← 0x30 AGENT_HANDSHAKE（実質は agent-card なので優先度低）
schema/agent-revoke.json        ← 0x35
schema/agent-ping.json          ← 0x36
schema/agent-log.json           ← 0x37
.github/ISSUE_TEMPLATE/         ← spec-proposal.md
scripts/validate.mjs            ← 全スキーマ×全サンプルをまとめて検証する Node スクリプト
```

新規ファイルを作る際は、`MINATO_PROTOCOL.md` の該当章と整合していることを必ず確認してください。
Agent Card / 各メッセージの `agent_id` や `signature` のパターンは `schema/common.json` に集約されています。

## アーキテクチャ（big picture）

MINATO は「Bitchat の Transport 層の上に、エージェント専用のメッセージタイプとセマンティクスを足したもの」と捉えると理解しやすいです。
Bitchat を fork して **新規コードだけを `MINATO/` サブディレクトリに追加** する方針で、既存の BLE / Nostr / Noise 実装はそのまま流用します。

```
Application Layer          SwiftUI / Jetpack Compose
Agent Layer                AgentCore / PermissionManager / CalendarAdapter / AIEngine   ← MINATO が追加
MINATO Protocol            0x30–0x37 の 8 メッセージタイプ                              ← MINATO が追加
Bitchat Protocol           既存の Transport メッセージ                                   ← 流用
Transport                  BLE Mesh（近距離）／ Nostr Relay（遠距離）自動切替          ← 流用
Crypto                     Noise Protocol / Curve25519 / Ed25519 / ChaCha20-Poly1305    ← 流用
```

### 4つのコアコンセプト

仕様を読み書きする際、以下 4 つが互いに参照し合うことを意識してください。
詳細は `MINATO_PROTOCOL.md` §4〜§9 を参照。

1. **Agent Card**（§4）— エージェントの「名刺」。ハンドシェイク時に一度だけ交換される宣言的な JSON で、`capabilities`, `default_trust_mode`, `supported_intents`, `ai_engine` を含む。Ed25519 署名付き。
2. **Message Types**（§5）— `0x30 AGENT_HANDSHAKE` から `0x37 AGENT_LOG` までの 8 種。Bitchat の既存コードに新規 enum として追加する形で実装される。
3. **Trust Mode**（§7）— `plan`（見習い）/ `suggest`（相棒）/ `auto`（右腕）/ `full_auto`（分身）。相手 npub ごとにローカル保存され、自動実行の粒度を決定する。Claude Code の実行モードが設計の参考。
4. **Capabilities × Intents**（§8, §9）— Capability は「何ができるか」の静的宣言（Agent Card に記載）、Intent は「このメッセージは何を意図しているか」の動的ラベル（payload に記載）。たとえば `schedule.write` Capability を持つエージェントは `schedule.confirm` Intent を処理できる。

### 多言語ペイロードの設計意図（§11）

`content` は送信者の母語、`translated_content` は受信者言語への翻訳を同じ payload に両方含めます。
これは「翻訳を transport の外側（AI エンジン）に置くことで、プロトコルは言語非依存のままにする」という設計です。
サンプルや仕様を追加する際、この両フィールドを常にセットで扱ってください。

### Transport の自動切替

`BLE 検出 → BLE Mesh / それ以外 → Nostr Relay (NIP-17)`。
MINATO のメッセージは transport に非依存で、同じ JSON payload がどちらでも流れます。
仕様を書く際、transport 固有の挙動（再送、fragmentation 等）は Bitchat に委ね、MINATO 層では触れないのが原則です。

## 他リポジトリとの関係

| リポジトリ | 役割 | 備考 |
|-----------|------|------|
| `minato-spec`（ここ） | 仕様・スキーマ・サンプルの正典 | Markdown + JSON のみ |
| `minato-ios` | iOS 実装 | `jackjackbits/bitchat` を fork、`MINATO/` を追加 |
| `minato-android` | Android 実装 | `permissionlesstech/bitchat-android` を fork、`minato/` を追加 |

実装側の詳細タスクは各リポジトリの CLAUDE.md に置きます。このファイルには書かないでください。

## iOS ビルド・デプロイ運用（最優先ルール）

**ビルド・インストール・テストは全て CLI で実行する。Xcode GUI の手動操作は最終手段。**

```bash
# ビルド
cd /Users/aikirishimaphoenix/minato-ios
xcodebuild -project bitchat.xcodeproj -scheme "bitchat (iOS)" \
  -destination 'generic/platform=iOS' -quiet build

# 両デバイスに同時インストール
APP_PATH=$(ls -d ~/Library/Developer/Xcode/DerivedData/bitchat-*/Build/Products/Debug-iphoneos/bitchat.app)
xcrun devicectl device install app --device 00008110-0012152A2E9A201E "$APP_PATH" &
xcrun devicectl device install app --device 00008103-000A055636FB001E "$APP_PATH" &
wait
```

- コード変更 → ビルド → デバイスインストールまで一気通貫で Claude が実行
- ユーザーに「Xcode で開いて Run して」と頼まない
- Xcode GUI が必要なのは Signing 設定変更など本当に手動でしかできない場合のみ

## コーディング / ドキュメント規約

- **JSON フィールド名**: `snake_case`（例: `minato_version`, `default_trust_mode`, `translated_content`）。仕様・スキーマ・サンプル全てで統一。
- **Trust Mode の技術名**: `plan` / `suggest` / `auto` / `full_auto`。日本語表示名（見習い / 相棒 / 右腕 / 分身）は UI 用の別レイヤー。`MINATO_PROTOCOL.md` §7 と README が正。
- **時刻**: ISO 8601（`2026-04-17T19:00:00+09:00`）または Unix timestamp（`created_at` など int）。仕様中の既存の表記に合わせる。
- **コミットメッセージ**: `feat:` / `fix:` / `docs:` / `chore:` の prefix を付与。
- **ブランチ**: `feature/xxx` / `fix/xxx`。
- **仕様のバージョン**: `MINATO_PROTOCOL.md` 冒頭と `minato_version` フィールドが現状 `0.1`。破壊的変更時はここを同時に更新。

## 参照リポジトリ

| 用途 | URL |
|-----|-----|
| Bitchat iOS（ベース） | https://github.com/jackjackbits/bitchat |
| Bitchat Android（ベース） | https://github.com/permissionlesstech/bitchat-android |
| Nostr NIPs（特に NIP-17） | https://github.com/nostr-protocol/nips |
| Noise Protocol Framework | https://noiseprotocol.org/ |

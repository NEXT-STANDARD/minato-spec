# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## このリポジトリについて

`minato-spec` は **MINATO Agent Protocol の仕様リポジトリ** です。
ソースコードは含まれず、仕様書・JSON Schema・サンプルペイロード・ガバナンス文書のみを管理します。
実装は別リポジトリ（`minato-ios`, `minato-android`）で行われる予定です。

仕様本体は `MINATO_PROTOCOL.md` が単一の正典（source of truth）です。
変更する際は必ず `MINATO_PROTOCOL.md` を最初に更新し、派生ファイル（スキーマ、サンプル、README）をそれに追従させてください。

## ビルド / テスト

このリポジトリにはビルド・テスト・リントのコマンドはありません（Markdown と JSON のみ）。
将来 `schema/` ディレクトリが作成された場合、JSON Schema の妥当性検証は `ajv` 等で
各 `examples/*.json` に対して行うことを想定しています（未整備）。

## 現在のディレクトリ状態

```
minato-spec/
├── CLAUDE.md           ← このファイル
├── LICENSE             ← MIT
├── MINATO_PROTOCOL.md  ← 仕様本体（v0.1 Draft）
└── README.md           ← プロジェクト紹介
```

以下は `MINATO_PROTOCOL.md` の §13「実装ガイドライン」および README の Roadmap で予定されているが **未作成** のディレクトリです:

```
docs/ja/, docs/en/              ← 仕様の人間向け解説
schema/agent-card.json          ← Agent Card の JSON Schema
schema/agent-message.json       ← AGENT_MESSAGE の JSON Schema
schema/agent-request.json       ← AGENT_REQUEST の JSON Schema
examples/                       ← handshake / schedule-negotiate / multilingual
.github/ISSUE_TEMPLATE/         ← spec-proposal.md
```

新規ファイルを作る際は、`MINATO_PROTOCOL.md` の該当章と整合していることを必ず確認してください。

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

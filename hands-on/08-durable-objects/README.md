# ハンズオン08: Durable Objectsでリアルタイム通信

## 学習目標
- Durable Objectsの基本概念
- WebSocketを使ったリアルタイム通信
- ステートフルなアプリケーション
- チャットアプリケーションの構築

## カリキュラム構成

### [Step 1: Durable Objectsの基礎](./step1-basics.md)
- Durable Objectsとは
- 基本的な実装
- Storageの使用

### [Step 2: WebSocket通信](./step2-websocket.md)
- WebSocketの接続
- メッセージの送受信
- 接続管理

### [Step 3: リアルタイムチャット](./step3-realtime-chat.md)
- チャットルームの実装
- ブロードキャスト機能
- ユーザー管理

### [Step 4: 実践アプリケーション](./step4-practical-app.md)
- 協調編集ツール
- ゲームのロビー
- リアルタイムダッシュボード

## 事前準備

\`\`\`bash
mkdir durable-objects-demo
cd durable-objects-demo
wrangler init
\`\`\`

## wrangler.toml設定

\`\`\`toml
name = "durable-objects-demo"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[[durable_objects.bindings]]
name = "CHAT_ROOM"
class_name = "ChatRoom"

[[migrations]]
tag = "v1"
new_classes = ["ChatRoom"]
\`\`\`

👉 [Step 1: Durable Objectsの基礎](./step1-basics.md)

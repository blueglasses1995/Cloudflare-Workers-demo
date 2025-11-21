# Cloudflare Workers 完全カリキュラム

Cloudflare Workersの基礎から実践まで学べる包括的な学習教材です。

## 📚 カリキュラム構成

### 1. 理論編
- [01. Cloudflare Workers概要](./docs/01-overview.md)
  - 技術的原理
  - アーキテクチャ
  - 思想と設計哲学
  - 歴史と発展

- [02. 公式仕様と技術詳細](./docs/02-specifications.md)
  - V8 Isolates
  - Runtime API
  - パフォーマンス特性
  - セキュリティモデル

- [03. 機能一覧と詳細](./docs/03-features.md)
  - コア機能
  - Workers KV
  - Durable Objects
  - R2 Storage
  - D1 Database
  - Queue
  - Analytics Engine
  - その他のバインディング

### 2. ハンズオン編

#### 基礎編
- [ハンズオン01: 環境構築とHello World](./hands-on/01-getting-started/README.md)
- [ハンズオン02: リクエスト/レスポンス処理](./hands-on/02-request-response/README.md)
- [ハンズオン03: ルーティングの実装](./hands-on/03-routing/README.md)

#### 実践編
- [ハンズオン04: Workers KVでのデータ管理](./hands-on/04-workers-kv/README.md)
- [ハンズオン05: REST API開発](./hands-on/05-rest-api/README.md)
- [ハンズオン06: 外部API連携](./hands-on/06-external-api/README.md)
- [ハンズオン07: 認証の実装](./hands-on/07-authentication/README.md)

#### 応用編
- [ハンズオン08: Durable Objectsでリアルタイム通信](./hands-on/08-durable-objects/README.md)
- [ハンズオン09: R2でファイルストレージ](./hands-on/09-r2-storage/README.md)
- [ハンズオン10: D1データベース活用](./hands-on/10-d1-database/README.md)
- [ハンズオン11: フルスタックアプリケーション](./hands-on/11-fullstack-app/README.md)

### 3. 実務編
- [デプロイメント戦略](./docs/04-deployment.md)
- [モニタリングとデバッグ](./docs/05-monitoring.md)
- [パフォーマンス最適化](./docs/06-performance.md)
- [セキュリティベストプラクティス](./docs/07-security.md)

## 🎯 学習の進め方

1. **理論編を読む**: まず概要と仕様を理解する
2. **基礎編から順番に実施**: 各ハンズオンを順番に進める
3. **実践編で応用力を養う**: 実務に近い課題に取り組む
4. **応用編で高度な機能を習得**: 最新機能を活用する

## 📋 前提知識

- JavaScript/TypeScript の基本的な知識
- HTTP プロトコルの基礎理解
- コマンドラインの基本操作
- Git の基本操作

## 🛠️ 必要な環境

- Node.js 16.13.0 以上
- npm または yarn
- Cloudflare アカウント（無料プランで可）
- テキストエディタ（VS Code推奨）

## 🚀 クイックスタート

```bash
# Wranglerのインストール
npm install -g wrangler

# Cloudflareへログイン
wrangler login

# プロジェクト作成
wrangler init my-worker

# ローカル開発サーバー起動
wrangler dev

# デプロイ
wrangler deploy
```

## 📖 各ハンズオンの構成

各ハンズオンは以下の構成で統一されています：

1. **学習目標**: 何を学ぶか
2. **事前準備**: 必要な知識とツール
3. **実装手順**: ステップバイステップのガイド
4. **コード解説**: 重要なコンセプトの説明
5. **動作確認**: テストとデバッグ方法
6. **デプロイ**: 本番環境への展開
7. **演習問題**: 理解を深めるための課題
8. **まとめ**: 学んだことの振り返り

## 🌟 このカリキュラムの特徴

- **包括的**: 基礎から応用まで網羅
- **実践的**: 実務で使える実装例
- **最新**: 2024年時点の最新機能に対応
- **日本語**: すべて日本語で解説
- **ハンズオン重視**: 実際に手を動かして学ぶ

## 📚 参考リソース

- [Cloudflare Workers公式ドキュメント](https://developers.cloudflare.com/workers/)
- [Cloudflare Developers Discord](https://discord.gg/cloudflaredev)
- [Wrangler CLI ドキュメント](https://developers.cloudflare.com/workers/wrangler/)

## 🤝 貢献

このカリキュラムの改善提案や誤りの修正は歓迎します。

## 📄 ライセンス

MIT License

---

**最終更新**: 2024年11月

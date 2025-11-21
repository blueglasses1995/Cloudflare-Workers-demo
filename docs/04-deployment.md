# 04. デプロイメント戦略

## デプロイの基本

### 基本的なデプロイ

```bash
wrangler deploy
```

### デプロイ前のチェック

```bash
# 構文チェック
npm run build

# TypeScriptの型チェック
npm run type-check

# ローカルテスト
wrangler dev
```

## 環境管理

### 環境の分離

```toml
# wrangler.toml

# 開発環境
[env.dev]
name = "my-worker-dev"
vars = { ENVIRONMENT = "development" }

# ステージング環境
[env.staging]
name = "my-worker-staging"
vars = { ENVIRONMENT = "staging" }

# 本番環境
[env.production]
name = "my-worker-production"
vars = { ENVIRONMENT = "production" }
```

### 環境ごとのデプロイ

```bash
# 開発環境
wrangler deploy --env dev

# ステージング環境
wrangler deploy --env staging

# 本番環境
wrangler deploy --env production
```

## CI/CDパイプライン

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy to Cloudflare Workers

on:
  push:
    branches:
      - main
      - develop

jobs:
  deploy:
    runs-on: ubuntu-latest
    name: Deploy
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Deploy to staging
        if: github.ref == 'refs/heads/develop'
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          environment: 'staging'

      - name: Deploy to production
        if: github.ref == 'refs/heads/main'
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          environment: 'production'
```

## カスタムドメイン

### ドメインの設定

1. Cloudflareダッシュボードでドメインを追加
2. Workers & Pagesでカスタムドメインを設定

```toml
# wrangler.toml
routes = [
  { pattern = "api.example.com/*", zone_name = "example.com" }
]
```

## ロールバック戦略

### バージョン管理

```bash
# デプロイ履歴確認
wrangler deployments list

# 特定のバージョンにロールバック
wrangler rollback [deployment-id]
```

## ベストプラクティス

1. **環境変数の管理**: Secretsを使用
2. **段階的デプロイ**: dev → staging → production
3. **自動テスト**: CI/CDでテスト実行
4. **モニタリング**: デプロイ後の監視
5. **ロールバック計画**: 問題発生時の対応手順

---

次: [05. モニタリングとデバッグ](./05-monitoring.md)

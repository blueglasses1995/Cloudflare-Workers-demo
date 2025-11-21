# ハンズオン01: 環境構築とHello World

## 学習目標

- Cloudflare Workers開発環境のセットアップ
- Wrangler CLIの基本操作
- 初めてのWorkerの作成とデプロイ
- ローカル開発サーバーの使用

## 事前準備

### 必要なもの

- Node.js 16.13.0 以上
- npm または yarn
- Cloudflareアカウント（無料）
- テキストエディタ（VS Code推奨）

### Node.jsのインストール確認

```bash
node --version
npm --version
```

## Step 1: Cloudflareアカウントの作成

1. https://dash.cloudflare.com/sign-up にアクセス
2. メールアドレスとパスワードを入力
3. メール認証を完了

## Step 2: Wrangler CLIのインストール

Wranglerは、Cloudflare Workers開発のための公式CLIツールです。

```bash
# グローバルインストール
npm install -g wrangler

# バージョン確認
wrangler --version
```

## Step 3: Cloudflareへのログイン

```bash
wrangler login
```

ブラウザが開き、Cloudflareへのアクセス許可を求められます。許可してください。

## Step 4: 初めてのWorkerプロジェクト作成

### プロジェクトの作成

```bash
# プロジェクトディレクトリを作成
mkdir hello-worker
cd hello-worker

# Wranglerプロジェクトの初期化
wrangler init

# 対話的な質問に答える：
# - Would you like to use TypeScript? → Yes
# - Would you like to use git? → Yes
# - Would you like to install wrangler into package.json? → Yes
# - Would you like to create a Worker? → Fetch handler
```

### 生成されたファイル構成

```
hello-worker/
├── src/
│   └── index.ts        # Workerのメインコード
├── .gitignore
├── package.json
├── tsconfig.json
├── wrangler.toml       # Wrangler設定ファイル
└── README.md
```

## Step 5: コードの確認と編集

### src/index.ts

生成されたコードを確認：

```typescript
export interface Env {
  // 環境変数の型定義をここに追加
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    return new Response('Hello World!');
  },
};
```

### コードの説明

- **export default**: Workerのエントリーポイント
- **fetch()**: HTTPリクエストを処理する関数
  - `request`: リクエスト情報
  - `env`: 環境変数とバインディング
  - `ctx`: 実行コンテキスト
- **Response**: HTTPレスポンスを返す

### より実用的なコードに変更

```typescript
export interface Env {
  // 環境変数
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);

    // JSON レスポンス
    if (url.pathname === '/json') {
      return new Response(
        JSON.stringify({
          message: 'Hello from Cloudflare Workers!',
          timestamp: new Date().toISOString(),
          method: request.method,
          url: request.url,
        }),
        {
          headers: {
            'Content-Type': 'application/json',
          },
        }
      );
    }

    // HTML レスポンス
    if (url.pathname === '/html') {
      const html = `
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Hello Worker</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      max-width: 800px;
      margin: 50px auto;
      padding: 20px;
    }
    h1 { color: #f38020; }
    .info { background: #f0f0f0; padding: 15px; border-radius: 5px; }
  </style>
</head>
<body>
  <h1>Hello from Cloudflare Workers!</h1>
  <div class="info">
    <p><strong>Request Method:</strong> ${request.method}</p>
    <p><strong>Request URL:</strong> ${request.url}</p>
    <p><strong>Timestamp:</strong> ${new Date().toISOString()}</p>
  </div>
</body>
</html>
      `.trim();

      return new Response(html, {
        headers: {
          'Content-Type': 'text/html; charset=utf-8',
        },
      });
    }

    // デフォルトレスポンス
    return new Response('Hello World!');
  },
};
```

## Step 6: ローカル開発サーバーで実行

```bash
wrangler dev
```

これにより、ローカル開発サーバーが起動します（通常は http://localhost:8787）。

### 動作確認

ブラウザまたはcurlで以下のURLにアクセス：

```bash
# プレーンテキスト
curl http://localhost:8787/

# JSON
curl http://localhost:8787/json

# HTML
open http://localhost:8787/html  # macOS
# または
start http://localhost:8787/html  # Windows
```

### ホットリロード

ファイルを編集すると、自動的にWorkerがリロードされます。試しにコードを変更してみましょう。

## Step 7: wrangler.tomlの設定

`wrangler.toml`を開き、設定を確認：

```toml
name = "hello-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"

# アカウント情報（オプション）
# account_id = "your-account-id"

# ルート設定
# routes = [
#   { pattern = "example.com/*", zone_name = "example.com" }
# ]
```

**主要な設定項目**:
- `name`: Workerの名前
- `main`: エントリーポイント
- `compatibility_date`: 互換性日付
- `account_id`: CloudflareアカウントID
- `routes`: カスタムドメインのルーティング

## Step 8: Cloudflareへデプロイ

### デプロイコマンド

```bash
wrangler deploy
```

### デプロイ結果

```
✨ Compiled Worker successfully
✨ Uploading...
✨ Deployment complete! Take a look over at https://hello-worker.your-subdomain.workers.dev
```

デプロイされたURLにアクセスして動作を確認しましょう。

### デプロイ先の確認

Cloudflareダッシュボードで確認：
1. https://dash.cloudflare.com/ にアクセス
2. 左メニューから「Workers & Pages」を選択
3. デプロイしたWorkerをクリック

## Step 9: ログの確認

### リアルタイムログの表示

```bash
wrangler tail
```

デプロイされたWorkerにリクエストを送ると、リアルタイムでログが表示されます。

### ログ出力の追加

```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // ログ出力
    console.log('Request received:', {
      method: request.method,
      url: request.url,
      headers: Object.fromEntries(request.headers),
    });

    return new Response('Hello World!');
  },
};
```

デプロイ後、`wrangler tail`でログを確認できます。

## Step 10: 環境変数の設定

### wrangler.tomlで環境変数を定義

```toml
[vars]
ENVIRONMENT = "production"
API_VERSION = "v1"
```

### コードで環境変数を使用

```typescript
export interface Env {
  ENVIRONMENT: string;
  API_VERSION: string;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    return new Response(
      JSON.stringify({
        environment: env.ENVIRONMENT,
        apiVersion: env.API_VERSION,
      }),
      {
        headers: {
          'Content-Type': 'application/json',
        },
      }
    );
  },
};
```

## Step 11: シークレットの管理

機密情報（APIキー等）はシークレットとして管理します。

### シークレットの設定

```bash
wrangler secret put API_KEY
# プロンプトが表示されたら、APIキーを入力
```

### コードでシークレットを使用

```typescript
export interface Env {
  API_KEY: string;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // env.API_KEY からアクセス（値はコードに含まれない）
    const apiKey = env.API_KEY;

    // 外部APIへのリクエスト例
    const response = await fetch('https://api.example.com/data', {
      headers: {
        'Authorization': `Bearer ${apiKey}`,
      },
    });

    return response;
  },
};
```

## よくあるコマンド

```bash
# ローカル開発
wrangler dev

# デプロイ
wrangler deploy

# ログ確認
wrangler tail

# シークレット設定
wrangler secret put SECRET_NAME

# シークレット一覧
wrangler secret list

# 削除
wrangler delete hello-worker
```

## トラブルシューティング

### エラー: "No account_id found"

**解決策**:
```bash
# アカウントIDを確認
wrangler whoami

# wrangler.tomlに追加
account_id = "your-account-id"
```

### エラー: "CPU time limit exceeded"

**解決策**: 処理が重すぎます。最適化するか、有料プランに変更してください。

### ローカルサーバーが起動しない

**解決策**:
```bash
# ポートが使用中の場合
wrangler dev --port 8788

# キャッシュをクリア
wrangler dev --local
```

## 演習問題

### 問題1: パラメータ付きレスポンス

URLパラメータ`name`を受け取り、「Hello, {name}!」と返すWorkerを作成してください。

**例**: `/greet?name=Alice` → `Hello, Alice!`

<details>
<summary>解答例</summary>

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const name = url.searchParams.get('name') || 'World';

    return new Response(`Hello, ${name}!`);
  },
};
```
</details>

### 問題2: 時刻に応じた挨拶

現在の時刻に応じて、「おはよう」「こんにちは」「こんばんは」を返すWorkerを作成してください。

<details>
<summary>解答例</summary>

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const hour = new Date().getHours();

    let greeting;
    if (hour < 12) {
      greeting = 'おはようございます';
    } else if (hour < 18) {
      greeting = 'こんにちは';
    } else {
      greeting = 'こんばんは';
    }

    return new Response(greeting);
  },
};
```
</details>

### 問題3: リクエスト情報の表示

リクエストヘッダーをすべてJSON形式で返すWorkerを作成してください。

<details>
<summary>解答例</summary>

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const headers: Record<string, string> = {};

    for (const [key, value] of request.headers) {
      headers[key] = value;
    }

    return new Response(JSON.stringify(headers, null, 2), {
      headers: {
        'Content-Type': 'application/json',
      },
    });
  },
};
```
</details>

## まとめ

このハンズオンで学んだこと：

✅ Wrangler CLIのインストールとセットアップ
✅ 初めてのWorkerの作成
✅ ローカル開発サーバーの使用
✅ Cloudflareへのデプロイ
✅ 環境変数とシークレットの管理
✅ ログの確認方法

## 次のステップ

次のハンズオンでは、リクエスト/レスポンス処理の詳細を学びます。

- [ハンズオン02: リクエスト/レスポンス処理](../02-request-response/README.md)

## 参考リソース

- [Wrangler CLI ドキュメント](https://developers.cloudflare.com/workers/wrangler/)
- [Workers Get Started Guide](https://developers.cloudflare.com/workers/get-started/)

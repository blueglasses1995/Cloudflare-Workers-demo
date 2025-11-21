# Step 1: 基本的なルーティング

## このステップで学ぶこと

- URLパスの解析
- 静的ルートのマッチング
- HTTPメソッドの判定
- 基本的なレスポンス返却

## Step 1-1: プロジェクトのセットアップ

まず、新しいWorkerプロジェクトを作成します。

```bash
mkdir routing-demo
cd routing-demo
wrangler init

# TypeScript、Git、パッケージマネージャーを選択
```

## Step 1-2: 最もシンプルなルーティング

### src/index.ts

まずは最もシンプルな形からスタートします：

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;

    // パスで分岐
    if (path === '/') {
      return new Response('Home Page');
    }

    if (path === '/about') {
      return new Response('About Page');
    }

    if (path === '/contact') {
      return new Response('Contact Page');
    }

    // マッチしない場合は404
    return new Response('Not Found', { status: 404 });
  },
};
```

### 動作確認

```bash
wrangler dev
```

ブラウザまたはcurlでテスト：

```bash
curl http://localhost:8787/
# => Home Page

curl http://localhost:8787/about
# => About Page

curl http://localhost:8787/contact
# => Contact Page

curl http://localhost:8787/unknown
# => Not Found
```

## Step 1-3: HTTPメソッドの処理

次に、HTTPメソッド（GET, POST等）も考慮します：

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;
    const method = request.method;

    // GET /
    if (method === 'GET' && path === '/') {
      return new Response('Home Page');
    }

    // GET /about
    if (method === 'GET' && path === '/about') {
      return new Response('About Page');
    }

    // POST /contact
    if (method === 'POST' && path === '/contact') {
      const body = await request.json();
      return new Response(
        JSON.stringify({
          message: 'Contact form submitted',
          data: body,
        }),
        {
          headers: { 'Content-Type': 'application/json' },
        }
      );
    }

    // GET /contact (フォーム表示)
    if (method === 'GET' && path === '/contact') {
      return new Response('Contact Form');
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

### 動作確認

```bash
# GET リクエスト
curl http://localhost:8787/contact
# => Contact Form

# POST リクエスト
curl -X POST http://localhost:8787/contact \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@example.com"}'
# => {"message":"Contact form submitted","data":{"name":"Alice","email":"alice@example.com"}}
```

## Step 1-4: ルーティングロジックの整理

if文が増えると読みにくくなるため、switch文を使って整理します：

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;
    const method = request.method;

    // ルートキーを作成（例: "GET /" or "POST /contact"）
    const routeKey = `${method} ${path}`;

    switch (routeKey) {
      case 'GET /':
        return handleHome(request);

      case 'GET /about':
        return handleAbout(request);

      case 'GET /contact':
        return handleContactForm(request);

      case 'POST /contact':
        return handleContactSubmit(request);

      case 'GET /api/status':
        return handleApiStatus(request);

      default:
        return handleNotFound(request);
    }
  },
};

// ハンドラー関数
function handleHome(request: Request): Response {
  return new Response('Home Page', {
    headers: { 'Content-Type': 'text/plain' },
  });
}

function handleAbout(request: Request): Response {
  return new Response('About Page', {
    headers: { 'Content-Type': 'text/plain' },
  });
}

function handleContactForm(request: Request): Response {
  const html = `
    <!DOCTYPE html>
    <html>
      <head><title>Contact</title></head>
      <body>
        <h1>Contact Form</h1>
        <form method="POST">
          <input name="name" placeholder="Name" required />
          <input name="email" type="email" placeholder="Email" required />
          <button type="submit">Submit</button>
        </form>
      </body>
    </html>
  `;
  return new Response(html, {
    headers: { 'Content-Type': 'text/html' },
  });
}

async function handleContactSubmit(request: Request): Promise<Response> {
  const body = await request.json();
  return new Response(
    JSON.stringify({
      message: 'Contact form submitted',
      data: body,
    }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
}

function handleApiStatus(request: Request): Response {
  return new Response(
    JSON.stringify({
      status: 'ok',
      timestamp: new Date().toISOString(),
    }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
}

function handleNotFound(request: Request): Response {
  return new Response('Not Found', {
    status: 404,
    headers: { 'Content-Type': 'text/plain' },
  });
}
```

## Step 1-5: オブジェクトベースのルーティング

さらに整理して、ルート定義をオブジェクトにまとめます：

```typescript
type RouteHandler = (request: Request) => Response | Promise<Response>;

const routes: Record<string, RouteHandler> = {
  'GET /': handleHome,
  'GET /about': handleAbout,
  'GET /contact': handleContactForm,
  'POST /contact': handleContactSubmit,
  'GET /api/status': handleApiStatus,
  'GET /api/time': handleApiTime,
};

export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;
    const method = request.method;
    const routeKey = `${method} ${path}`;

    // ルートを探す
    const handler = routes[routeKey];

    if (handler) {
      return handler(request);
    }

    return handleNotFound(request);
  },
};

// ハンドラー関数
function handleHome(request: Request): Response {
  return new Response('Home Page');
}

function handleAbout(request: Request): Response {
  return new Response('About Page');
}

function handleContactForm(request: Request): Response {
  return new Response('Contact Form');
}

async function handleContactSubmit(request: Request): Promise<Response> {
  const body = await request.json();
  return new Response(JSON.stringify({ message: 'Submitted', data: body }), {
    headers: { 'Content-Type': 'application/json' },
  });
}

function handleApiStatus(request: Request): Response {
  return new Response(
    JSON.stringify({ status: 'ok', timestamp: new Date().toISOString() }),
    { headers: { 'Content-Type': 'application/json' } }
  );
}

function handleApiTime(request: Request): Response {
  return new Response(
    JSON.stringify({ time: new Date().toISOString() }),
    { headers: { 'Content-Type': 'application/json' } }
  );
}

function handleNotFound(request: Request): Response {
  return new Response('Not Found', { status: 404 });
}
```

## 演習問題

### 問題1: 新しいルートの追加

以下のルートを追加してください：

- `GET /services` - "Our Services"を返す
- `GET /pricing` - "Pricing"を返す
- `GET /api/version` - `{"version": "1.0.0"}`をJSON形式で返す

<details>
<summary>解答例</summary>

```typescript
const routes: Record<string, RouteHandler> = {
  'GET /': handleHome,
  'GET /about': handleAbout,
  'GET /services': handleServices,
  'GET /pricing': handlePricing,
  'GET /api/version': handleApiVersion,
};

function handleServices(request: Request): Response {
  return new Response('Our Services');
}

function handlePricing(request: Request): Response {
  return new Response('Pricing');
}

function handleApiVersion(request: Request): Response {
  return new Response(
    JSON.stringify({ version: '1.0.0' }),
    { headers: { 'Content-Type': 'application/json' } }
  );
}
```
</details>

### 問題2: クエリパラメータの取得

`GET /search?q=cloudflare`というリクエストを受け取り、クエリパラメータ`q`の値を含むJSONレスポンスを返すハンドラーを作成してください。

<details>
<summary>解答例</summary>

```typescript
const routes: Record<string, RouteHandler> = {
  // ...他のルート
  'GET /search': handleSearch,
};

function handleSearch(request: Request): Response {
  const url = new URL(request.url);
  const query = url.searchParams.get('q') || '';

  return new Response(
    JSON.stringify({
      query,
      results: [`Result 1 for "${query}"`, `Result 2 for "${query}"`],
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
}
```

テスト：
```bash
curl "http://localhost:8787/search?q=cloudflare"
```
</details>

## まとめ

このステップで学んだこと：

✅ URLパスとHTTPメソッドの解析
✅ 静的ルートのマッチング
✅ switch文やオブジェクトを使ったルーティング
✅ ハンドラー関数の分離
✅ クエリパラメータの取得

## 次のステップ

静的ルートは理解できました。次は動的ルート（パスパラメータ）を学びます。

👉 [Step 2: 動的ルート](./step2-dynamic-routes.md)

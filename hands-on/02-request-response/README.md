# ハンズオン02: リクエスト/レスポンス処理

## 学習目標

- Request/Response APIの詳細な使い方
- HTTPメソッドの処理
- ヘッダーの操作
- ボディの処理（JSON, FormData, etc）
- CORSの実装

## 事前準備

```bash
mkdir request-response-worker
cd request-response-worker
wrangler init
```

## Step 1: HTTPメソッドの処理

### 基本的なメソッド処理

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    // パスルーティング
    if (url.pathname === '/api/data') {
      // メソッドに応じた処理
      switch (request.method) {
        case 'GET':
          return handleGet(request);
        case 'POST':
          return handlePost(request);
        case 'PUT':
          return handlePut(request);
        case 'DELETE':
          return handleDelete(request);
        case 'OPTIONS':
          return handleOptions(request);
        default:
          return new Response('Method Not Allowed', { status: 405 });
      }
    }

    return new Response('Not Found', { status: 404 });
  },
};

async function handleGet(request: Request): Promise<Response> {
  return new Response(
    JSON.stringify({
      message: 'GET request received',
      data: [
        { id: 1, name: 'Item 1' },
        { id: 2, name: 'Item 2' },
      ],
    }),
    {
      headers: {
        'Content-Type': 'application/json',
      },
    }
  );
}

async function handlePost(request: Request): Promise<Response> {
  try {
    const body = await request.json();

    return new Response(
      JSON.stringify({
        message: 'POST request received',
        received: body,
      }),
      {
        status: 201,
        headers: {
          'Content-Type': 'application/json',
        },
      }
    );
  } catch (error) {
    return new Response('Invalid JSON', { status: 400 });
  }
}

async function handlePut(request: Request): Promise<Response> {
  const body = await request.json();

  return new Response(
    JSON.stringify({
      message: 'PUT request received',
      updated: body,
    }),
    {
      headers: {
        'Content-Type': 'application/json',
      },
    }
  );
}

async function handleDelete(request: Request): Promise<Response> {
  return new Response(null, { status: 204 });
}

async function handleOptions(request: Request): Promise<Response> {
  return new Response(null, {
    headers: {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    },
  });
}
```

### テスト

```bash
# ローカル起動
wrangler dev

# 別のターミナルで
# GET
curl http://localhost:8787/api/data

# POST
curl -X POST http://localhost:8787/api/data \
  -H "Content-Type: application/json" \
  -d '{"name": "New Item"}'

# PUT
curl -X PUT http://localhost:8787/api/data \
  -H "Content-Type: application/json" \
  -d '{"id": 1, "name": "Updated Item"}'

# DELETE
curl -X DELETE http://localhost:8787/api/data
```

## Step 2: リクエストヘッダーの処理

### ヘッダーの読み取り

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    // 特定のヘッダーを取得
    const userAgent = request.headers.get('User-Agent');
    const contentType = request.headers.get('Content-Type');
    const authorization = request.headers.get('Authorization');

    // Cloudflare固有のヘッダー
    const cfRay = request.headers.get('CF-Ray');
    const cfConnectingIp = request.headers.get('CF-Connecting-IP');
    const cfCountry = request.headers.get('CF-IPCountry');

    // すべてのヘッダーをイテレーション
    const headers: Record<string, string> = {};
    for (const [key, value] of request.headers) {
      headers[key] = value;
    }

    return new Response(
      JSON.stringify(
        {
          userAgent,
          contentType,
          cfRay,
          cfConnectingIp,
          cfCountry,
          allHeaders: headers,
        },
        null,
        2
      ),
      {
        headers: {
          'Content-Type': 'application/json',
        },
      }
    );
  },
};
```

### カスタムヘッダーの検証

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    // APIキーの検証
    const apiKey = request.headers.get('X-API-Key');

    if (!apiKey || apiKey !== 'secret-key') {
      return new Response('Unauthorized', {
        status: 401,
        headers: {
          'WWW-Authenticate': 'API-Key',
        },
      });
    }

    return new Response('Authorized!');
  },
};
```

## Step 3: レスポンスヘッダーの設定

### 基本的なヘッダー設定

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const headers = new Headers();

    // Content-Type
    headers.set('Content-Type', 'application/json; charset=utf-8');

    // キャッシュ制御
    headers.set('Cache-Control', 'public, max-age=3600');

    // セキュリティヘッダー
    headers.set('X-Content-Type-Options', 'nosniff');
    headers.set('X-Frame-Options', 'DENY');
    headers.set('X-XSS-Protection', '1; mode=block');
    headers.set(
      'Strict-Transport-Security',
      'max-age=31536000; includeSubDomains'
    );

    // カスタムヘッダー
    headers.set('X-Powered-By', 'Cloudflare Workers');

    return new Response(JSON.stringify({ message: 'Hello' }), {
      headers,
    });
  },
};
```

### Cookieの設定

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const response = new Response('Cookie set!');

    // シンプルなCookie
    response.headers.set('Set-Cookie', 'session=abc123');

    // 詳細なCookie設定
    response.headers.append(
      'Set-Cookie',
      'user=john; Path=/; Max-Age=3600; HttpOnly; Secure; SameSite=Strict'
    );

    // 複数のCookie
    response.headers.append('Set-Cookie', 'lang=ja; Path=/');

    return response;
  },
};
```

## Step 4: リクエストボディの処理

### JSONボディの処理

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    if (request.method !== 'POST') {
      return new Response('Method Not Allowed', { status: 405 });
    }

    try {
      // JSONとしてパース
      const data = await request.json();

      // バリデーション
      if (!data.name || !data.email) {
        return new Response(
          JSON.stringify({
            error: 'name and email are required',
          }),
          {
            status: 400,
            headers: { 'Content-Type': 'application/json' },
          }
        );
      }

      // 処理
      return new Response(
        JSON.stringify({
          message: 'Data received',
          data: {
            name: data.name,
            email: data.email,
          },
        }),
        {
          status: 201,
          headers: { 'Content-Type': 'application/json' },
        }
      );
    } catch (error) {
      return new Response(
        JSON.stringify({
          error: 'Invalid JSON',
        }),
        {
          status: 400,
          headers: { 'Content-Type': 'application/json' },
        }
      );
    }
  },
};
```

### FormDataの処理

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    if (request.method !== 'POST') {
      return new Response('Method Not Allowed', { status: 405 });
    }

    const contentType = request.headers.get('Content-Type') || '';

    if (contentType.includes('application/x-www-form-urlencoded')) {
      const formData = await request.formData();

      const name = formData.get('name');
      const email = formData.get('email');

      return new Response(
        JSON.stringify({
          name,
          email,
        }),
        {
          headers: { 'Content-Type': 'application/json' },
        }
      );
    }

    return new Response('Unsupported Media Type', { status: 415 });
  },
};
```

### テキストボディの処理

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const text = await request.text();

    return new Response(`Received: ${text.length} characters`, {
      headers: { 'Content-Type': 'text/plain' },
    });
  },
};
```

### バイナリデータの処理

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const arrayBuffer = await request.arrayBuffer();
    const bytes = new Uint8Array(arrayBuffer);

    return new Response(
      JSON.stringify({
        size: bytes.length,
        firstByte: bytes[0],
      }),
      {
        headers: { 'Content-Type': 'application/json' },
      }
    );
  },
};
```

## Step 5: CORS対応

### 基本的なCORS設定

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    // OPTIONSリクエスト（プリフライト）
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
          'Access-Control-Allow-Headers': 'Content-Type, Authorization',
          'Access-Control-Max-Age': '86400',
        },
      });
    }

    // 実際のリクエスト処理
    const response = await handleRequest(request);

    // CORSヘッダーを追加
    response.headers.set('Access-Control-Allow-Origin', '*');
    response.headers.set(
      'Access-Control-Allow-Methods',
      'GET, POST, PUT, DELETE, OPTIONS'
    );

    return response;
  },
};

async function handleRequest(request: Request): Promise<Response> {
  return new Response(
    JSON.stringify({ message: 'Hello with CORS' }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
}
```

### より安全なCORS設定

```typescript
const ALLOWED_ORIGINS = [
  'https://example.com',
  'https://app.example.com',
];

export default {
  async fetch(request: Request): Promise<Response> {
    const origin = request.headers.get('Origin');

    // オリジンの検証
    const isAllowed = origin && ALLOWED_ORIGINS.includes(origin);

    if (request.method === 'OPTIONS') {
      if (!isAllowed) {
        return new Response(null, { status: 403 });
      }

      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': origin,
          'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
          'Access-Control-Allow-Headers': 'Content-Type, Authorization',
          'Access-Control-Allow-Credentials': 'true',
          'Access-Control-Max-Age': '86400',
        },
      });
    }

    const response = await handleRequest(request);

    if (isAllowed) {
      response.headers.set('Access-Control-Allow-Origin', origin);
      response.headers.set('Access-Control-Allow-Credentials', 'true');
    }

    return response;
  },
};

async function handleRequest(request: Request): Promise<Response> {
  return new Response(
    JSON.stringify({ message: 'Secure CORS' }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
}
```

## Step 6: リダイレクト

### 基本的なリダイレクト

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    // パスベースのリダイレクト
    if (url.pathname === '/old-page') {
      return Response.redirect('https://example.com/new-page', 301);
    }

    // 条件付きリダイレクト
    if (url.pathname === '/mobile') {
      const userAgent = request.headers.get('User-Agent') || '';
      const isMobile = /Mobile|Android|iPhone/i.test(userAgent);

      if (isMobile) {
        return Response.redirect('https://m.example.com', 302);
      }
    }

    return new Response('Home Page');
  },
};
```

### HTTPSへの強制リダイレクト

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    // HTTPをHTTPSにリダイレクト
    if (url.protocol === 'http:') {
      url.protocol = 'https:';
      return Response.redirect(url.toString(), 301);
    }

    return new Response('Secure connection');
  },
};
```

## Step 7: エラーハンドリング

### 包括的なエラーハンドリング

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    try {
      return await handleRequest(request);
    } catch (error) {
      console.error('Error:', error);

      return new Response(
        JSON.stringify({
          error: 'Internal Server Error',
          message: error instanceof Error ? error.message : 'Unknown error',
        }),
        {
          status: 500,
          headers: { 'Content-Type': 'application/json' },
        }
      );
    }
  },
};

async function handleRequest(request: Request): Promise<Response> {
  const url = new URL(request.url);

  if (url.pathname === '/error') {
    throw new Error('Something went wrong!');
  }

  return new Response('OK');
}
```

## 演習問題

### 問題1: シンプルなRESTful API

以下の仕様を満たすAPIを実装してください：

- `GET /users` - ユーザー一覧を返す
- `POST /users` - ユーザーを作成（メモリ上に保存）
- `GET /users/:id` - 特定のユーザーを返す

<details>
<summary>解答例</summary>

```typescript
// メモリ上のストレージ（実際はKVなどを使用）
let users: any[] = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' },
];

export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const pathParts = url.pathname.split('/').filter(Boolean);

    // GET /users
    if (request.method === 'GET' && pathParts[0] === 'users' && !pathParts[1]) {
      return new Response(JSON.stringify(users), {
        headers: { 'Content-Type': 'application/json' },
      });
    }

    // GET /users/:id
    if (request.method === 'GET' && pathParts[0] === 'users' && pathParts[1]) {
      const id = parseInt(pathParts[1]);
      const user = users.find(u => u.id === id);

      if (!user) {
        return new Response('Not Found', { status: 404 });
      }

      return new Response(JSON.stringify(user), {
        headers: { 'Content-Type': 'application/json' },
      });
    }

    // POST /users
    if (request.method === 'POST' && pathParts[0] === 'users') {
      const body = await request.json();
      const newUser = {
        id: users.length + 1,
        name: body.name,
      };
      users.push(newUser);

      return new Response(JSON.stringify(newUser), {
        status: 201,
        headers: { 'Content-Type': 'application/json' },
      });
    }

    return new Response('Not Found', { status: 404 });
  },
};
```
</details>

### 問題2: リクエストログミドルウェア

すべてのリクエストのメソッド、パス、IPアドレスをログに記録する実装を追加してください。

### 問題3: レート制限

同一IPからのリクエストを1分間に10回までに制限する実装をしてください。

## まとめ

このハンズオンで学んだこと：

✅ HTTPメソッドの処理
✅ ヘッダーの読み取りと設定
✅ リクエストボディの各種形式の処理
✅ CORS対応
✅ リダイレクト
✅ エラーハンドリング

## 次のステップ

- [ハンズオン03: ルーティングの実装](../03-routing/README.md)

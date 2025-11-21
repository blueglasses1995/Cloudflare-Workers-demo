# Step 3: ミドルウェアパターン

## このステップで学ぶこと

- ミドルウェアの概念
- リクエスト前処理
- レスポンス後処理
- ミドルウェアチェーン
- エラーハンドリングミドルウェア

## Step 3-1: ミドルウェアとは

ミドルウェアは、リクエストとレスポンスの間で実行される処理です。

```
Request → Middleware 1 → Middleware 2 → Handler → Middleware 2 → Middleware 1 → Response
```

用途：
- ログ記録
- 認証・認可
- リクエストの変換
- レスポンスの変換
- エラーハンドリング

## Step 3-2: 最初のミドルウェア

### ログミドルウェア

```typescript
type Handler = (request: Request) => Response | Promise<Response>;
type Middleware = (
  request: Request,
  next: Handler
) => Response | Promise<Response>;

// ログミドルウェア
const loggingMiddleware: Middleware = async (request, next) => {
  const start = Date.now();
  const url = new URL(request.url);

  console.log(`→ ${request.method} ${url.pathname}`);

  const response = await next(request);

  const duration = Date.now() - start;
  console.log(`← ${response.status} ${url.pathname} (${duration}ms)`);

  return response;
};

// ハンドラー
async function handleUser(request: Request): Promise<Response> {
  return new Response('User Page');
}

// ミドルウェアを適用
export default {
  async fetch(request: Request): Promise<Response> {
    return loggingMiddleware(request, handleUser);
  },
};
```

### 動作確認

```bash
wrangler dev

# 別ターミナルで
curl http://localhost:8787/

# ログが表示される：
# → GET /
# ← 200 / (5ms)
```

## Step 3-3: 認証ミドルウェア

### API Key認証

```typescript
interface Env {
  API_KEY: string;
}

const authMiddleware: Middleware = async (request, next) => {
  const apiKey = request.headers.get('X-API-Key');

  if (!apiKey || apiKey !== 'secret-key-123') {
    return new Response('Unauthorized', {
      status: 401,
      headers: {
        'WWW-Authenticate': 'API-Key',
      },
    });
  }

  // 認証成功 - 次の処理へ
  return next(request);
};

async function handleProtectedResource(request: Request): Promise<Response> {
  return new Response('Protected Resource');
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // 認証ミドルウェアを適用
    return authMiddleware(request, handleProtectedResource);
  },
};
```

### 動作確認

```bash
# 認証なし - 401エラー
curl http://localhost:8787/
# => Unauthorized

# 認証あり - 成功
curl http://localhost:8787/ \
  -H "X-API-Key: secret-key-123"
# => Protected Resource
```

## Step 3-4: 複数のミドルウェアを組み合わせる

### ミドルウェアチェーン

```typescript
// ミドルウェアを合成する関数
function compose(...middlewares: Middleware[]): Middleware {
  return (request, finalHandler) => {
    let index = 0;

    const next: Handler = async (req) => {
      if (index >= middlewares.length) {
        return finalHandler(req);
      }

      const middleware = middlewares[index++];
      return middleware(req, next);
    };

    return next(request);
  };
}

// 各種ミドルウェア
const loggingMiddleware: Middleware = async (request, next) => {
  console.log(`→ ${request.method} ${new URL(request.url).pathname}`);
  const response = await next(request);
  console.log(`← ${response.status}`);
  return response;
};

const authMiddleware: Middleware = async (request, next) => {
  const apiKey = request.headers.get('X-API-Key');

  if (!apiKey || apiKey !== 'secret-key-123') {
    return new Response('Unauthorized', { status: 401 });
  }

  return next(request);
};

const corsMiddleware: Middleware = async (request, next) => {
  if (request.method === 'OPTIONS') {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
        'Access-Control-Allow-Headers': 'Content-Type, X-API-Key',
      },
    });
  }

  const response = await next(request);

  const newHeaders = new Headers(response.headers);
  newHeaders.set('Access-Control-Allow-Origin', '*');

  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers: newHeaders,
  });
};

const timingMiddleware: Middleware = async (request, next) => {
  const start = Date.now();
  const response = await next(request);
  const duration = Date.now() - start;

  const newHeaders = new Headers(response.headers);
  newHeaders.set('X-Response-Time', `${duration}ms`);

  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers: newHeaders,
  });
};

// ハンドラー
async function handleRequest(request: Request): Promise<Response> {
  return new Response(
    JSON.stringify({ message: 'Hello World' }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
}

// すべてのミドルウェアを合成
const app = compose(
  loggingMiddleware,
  corsMiddleware,
  timingMiddleware,
  authMiddleware
);

export default {
  async fetch(request: Request): Promise<Response> {
    return app(request, handleRequest);
  },
};
```

実行順序：
```
Request
  ↓
loggingMiddleware (開始)
  ↓
corsMiddleware (OPTIONS処理 or 継続)
  ↓
timingMiddleware (開始)
  ↓
authMiddleware (認証チェック)
  ↓
handleRequest (実際の処理)
  ↓
authMiddleware (後処理)
  ↓
timingMiddleware (レスポンスタイム追加)
  ↓
corsMiddleware (CORSヘッダー追加)
  ↓
loggingMiddleware (ログ出力)
  ↓
Response
```

## Step 3-5: エラーハンドリングミドルウェア

```typescript
const errorHandlerMiddleware: Middleware = async (request, next) => {
  try {
    return await next(request);
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
};

// エラーをスローするハンドラー
async function handleError(request: Request): Promise<Response> {
  throw new Error('Something went wrong!');
}

const app = compose(
  errorHandlerMiddleware,
  loggingMiddleware
);

export default {
  async fetch(request: Request): Promise<Response> {
    return app(request, handleError);
  },
};
```

## Step 3-6: リクエストコンテキストの拡張

ミドルウェアでリクエストに情報を追加したい場合：

```typescript
// カスタムリクエスト型
interface RequestWithUser extends Request {
  user?: {
    id: string;
    name: string;
  };
}

const authMiddleware = async (
  request: RequestWithUser,
  next: Handler
): Promise<Response> => {
  const token = request.headers.get('Authorization')?.replace('Bearer ', '');

  if (!token) {
    return new Response('Unauthorized', { status: 401 });
  }

  // トークンからユーザー情報を取得（簡略化）
  request.user = {
    id: '123',
    name: 'Alice',
  };

  return next(request);
};

async function handleProfile(request: RequestWithUser): Promise<Response> {
  const user = request.user;

  if (!user) {
    return new Response('Unauthorized', { status: 401 });
  }

  return new Response(
    JSON.stringify({
      id: user.id,
      name: user.name,
    }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
}
```

## Step 3-7: 条件付きミドルウェア

特定のパスにのみミドルウェアを適用：

```typescript
function conditionalMiddleware(
  path: RegExp,
  middleware: Middleware
): Middleware {
  return async (request, next) => {
    const url = new URL(request.url);

    if (path.test(url.pathname)) {
      return middleware(request, next);
    }

    return next(request);
  };
}

// /api/* パスにのみ認証を適用
const apiAuthMiddleware = conditionalMiddleware(
  /^\/api\//,
  authMiddleware
);

const app = compose(
  loggingMiddleware,
  apiAuthMiddleware // /api/* のみ認証が必要
);

export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === '/') {
      return new Response('Public Home Page');
    }

    if (url.pathname === '/api/users') {
      return app(request, async () => {
        return new Response('Protected API');
      });
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

## 演習問題

### 問題1: レート制限ミドルウェア

シンプルなレート制限ミドルウェアを実装してください。
- メモリ上で各IPのリクエスト数をカウント
- 1分間に10リクエストまで許可
- 超過した場合は429エラーを返す

<details>
<summary>解答例</summary>

```typescript
const requestCounts = new Map<string, { count: number; resetAt: number }>();

const rateLimitMiddleware: Middleware = async (request, next) => {
  const ip = request.headers.get('CF-Connecting-IP') || 'unknown';
  const now = Date.now();

  const record = requestCounts.get(ip);

  if (record) {
    if (now > record.resetAt) {
      // リセット時刻を過ぎたら初期化
      requestCounts.set(ip, { count: 1, resetAt: now + 60000 });
    } else if (record.count >= 10) {
      // レート制限超過
      return new Response('Too Many Requests', {
        status: 429,
        headers: {
          'Retry-After': '60',
        },
      });
    } else {
      // カウント増加
      record.count++;
    }
  } else {
    // 初回リクエスト
    requestCounts.set(ip, { count: 1, resetAt: now + 60000 });
  }

  return next(request);
};
```
</details>

### 問題2: リクエストボディのバリデーションミドルウェア

POSTリクエストのJSONボディに必須フィールドが含まれているかチェックするミドルウェアを作成してください。

<details>
<summary>解答例</summary>

```typescript
function validateBodyMiddleware(requiredFields: string[]): Middleware {
  return async (request, next) => {
    if (request.method === 'POST' || request.method === 'PUT') {
      try {
        const body = await request.json();

        for (const field of requiredFields) {
          if (!(field in body)) {
            return new Response(
              JSON.stringify({
                error: `Missing required field: ${field}`,
              }),
              {
                status: 400,
                headers: { 'Content-Type': 'application/json' },
              }
            );
          }
        }

        // ボディを再度読めるように、Requestを再構築
        const newRequest = new Request(request.url, {
          method: request.method,
          headers: request.headers,
          body: JSON.stringify(body),
        });

        return next(newRequest);
      } catch (error) {
        return new Response(
          JSON.stringify({ error: 'Invalid JSON' }),
          {
            status: 400,
            headers: { 'Content-Type': 'application/json' },
          }
        );
      }
    }

    return next(request);
  };
}

// 使用例
const validateUserMiddleware = validateBodyMiddleware(['name', 'email']);
```
</details>

## まとめ

このステップで学んだこと：

✅ ミドルウェアの概念と実装
✅ リクエスト前処理とレスポンス後処理
✅ 複数ミドルウェアの合成
✅ エラーハンドリングミドルウェア
✅ 条件付きミドルウェア

## 次のステップ

最後に、これまで学んだことを統合して、再利用可能なRouterクラスを実装します。

👉 [Step 4: Routerクラスの実装](./step4-router-class.md)

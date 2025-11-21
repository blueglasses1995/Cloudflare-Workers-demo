# Step 4: Routerクラスの実装

## このステップで学ぶこと

- 再利用可能なRouterクラスの設計
- ルート登録の抽象化
- メソッドごとのヘルパーメソッド
- ミドルウェアの統合
- 実践的な使用例

## Step 4-1: 基本的なRouterクラス

これまで学んだ知識を統合して、使いやすいRouterクラスを作ります。

```typescript
type Handler = (
  request: Request,
  params: Record<string, string>
) => Response | Promise<Response>;

type Middleware = (
  request: Request,
  next: () => Promise<Response>
) => Response | Promise<Response>;

interface Route {
  method: string;
  pattern: RegExp;
  handler: Handler;
}

class Router {
  private routes: Route[] = [];
  private middlewares: Middleware[] = [];

  // ルート登録
  add(method: string, pattern: string | RegExp, handler: Handler): void {
    const regex =
      typeof pattern === 'string' ? this.pathToRegex(pattern) : pattern;

    this.routes.push({
      method,
      pattern: regex,
      handler,
    });
  }

  // GETメソッド専用
  get(pattern: string | RegExp, handler: Handler): void {
    this.add('GET', pattern, handler);
  }

  // POSTメソッド専用
  post(pattern: string | RegExp, handler: Handler): void {
    this.add('POST', pattern, handler);
  }

  // PUTメソッド専用
  put(pattern: string | RegExp, handler: Handler): void {
    this.add('PUT', pattern, handler);
  }

  // DELETEメソッド専用
  delete(pattern: string | RegExp, handler: Handler): void {
    this.add('DELETE', pattern, handler);
  }

  // パスパターンを正規表現に変換
  // /users/:id → /^\/users\/(?<id>[^\/]+)$/
  private pathToRegex(path: string): RegExp {
    const pattern = path
      .replace(/\//g, '\\/')
      .replace(/:(\w+)/g, '(?<$1>[^\\/]+)');

    return new RegExp(`^${pattern}$`);
  }

  // ミドルウェア追加
  use(middleware: Middleware): void {
    this.middlewares.push(middleware);
  }

  // リクエストを処理
  async handle(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;
    const method = request.method;

    // ルートを探す
    for (const route of this.routes) {
      if (route.method !== method) {
        continue;
      }

      const match = path.match(route.pattern);

      if (match) {
        const params = match.groups || {};

        // ミドルウェアを適用
        return this.runMiddlewares(request, () =>
          route.handler(request, params)
        );
      }
    }

    // ルートが見つからない
    return new Response('Not Found', { status: 404 });
  }

  // ミドルウェアを順番に実行
  private async runMiddlewares(
    request: Request,
    finalHandler: () => Promise<Response>
  ): Promise<Response> {
    let index = 0;

    const next = async (): Promise<Response> => {
      if (index >= this.middlewares.length) {
        return finalHandler();
      }

      const middleware = this.middlewares[index++];
      return middleware(request, next);
    };

    return next();
  }
}

export { Router };
```

## Step 4-2: Routerクラスの使用例

### 基本的な使い方

```typescript
import { Router } from './router';

const router = new Router();

// ルート登録
router.get('/', (request, params) => {
  return new Response('Home Page');
});

router.get('/about', (request, params) => {
  return new Response('About Page');
});

router.get('/users/:id', (request, params) => {
  return new Response(
    JSON.stringify({
      userId: params.id,
      name: `User ${params.id}`,
    }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
});

router.post('/users', async (request, params) => {
  const body = await request.json();

  return new Response(
    JSON.stringify({
      id: Date.now().toString(),
      ...body,
    }),
    {
      status: 201,
      headers: { 'Content-Type': 'application/json' },
    }
  );
});

// Worker
export default {
  async fetch(request: Request): Promise<Response> {
    return router.handle(request);
  },
};
```

### ミドルウェアの追加

```typescript
import { Router } from './router';

const router = new Router();

// ログミドルウェア
router.use(async (request, next) => {
  const start = Date.now();
  console.log(`→ ${request.method} ${new URL(request.url).pathname}`);

  const response = await next();

  const duration = Date.now() - start;
  console.log(`← ${response.status} (${duration}ms)`);

  return response;
});

// CORSミドルウェア
router.use(async (request, next) => {
  if (request.method === 'OPTIONS') {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
        'Access-Control-Allow-Headers': 'Content-Type',
      },
    });
  }

  const response = await next();
  const newHeaders = new Headers(response.headers);
  newHeaders.set('Access-Control-Allow-Origin', '*');

  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers: newHeaders,
  });
});

// ルート登録
router.get('/api/users', (request, params) => {
  return new Response(
    JSON.stringify([
      { id: '1', name: 'Alice' },
      { id: '2', name: 'Bob' },
    ]),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
});

export default {
  async fetch(request: Request): Promise<Response> {
    return router.handle(request);
  },
};
```

## Step 4-3: ルートグループ機能の追加

共通のプレフィックスを持つルートをグループ化します。

```typescript
class Router {
  // ... 既存のコード

  // ルートグループ
  group(prefix: string, callback: (router: Router) => void): void {
    const groupRouter = new Router();

    // グループのルートを登録
    callback(groupRouter);

    // プレフィックスを追加して親ルーターに登録
    for (const route of groupRouter.routes) {
      const newPattern = this.addPrefix(route.pattern, prefix);
      this.routes.push({
        method: route.method,
        pattern: newPattern,
        handler: route.handler,
      });
    }
  }

  private addPrefix(pattern: RegExp, prefix: string): RegExp {
    const source = pattern.source;
    // ^\/を削除して、プレフィックスを追加
    const newSource = source.replace(/^\^\\\//, `^\\\/${prefix.replace(/^\//, '')}\\/`);
    return new RegExp(newSource);
  }
}
```

### 使用例

```typescript
const router = new Router();

// /api グループ
router.group('/api', (api) => {
  api.get('/users', listUsers);
  api.post('/users', createUser);
  api.get('/users/:id', getUser);
  api.put('/users/:id', updateUser);
  api.delete('/users/:id', deleteUser);
});

// /admin グループ
router.group('/admin', (admin) => {
  admin.get('/dashboard', showDashboard);
  admin.get('/users', adminListUsers);
  admin.post('/users/:id/ban', banUser);
});

// 実際のパス:
// GET /api/users
// POST /api/users
// GET /api/users/:id
// GET /admin/dashboard
// など
```

## Step 4-4: 完全版Routerクラス

### src/router.ts

```typescript
type Handler = (
  request: Request,
  params: Record<string, string>
) => Response | Promise<Response>;

type Middleware = (
  request: Request,
  next: () => Promise<Response>
) => Response | Promise<Response>;

interface Route {
  method: string;
  pattern: RegExp;
  handler: Handler;
}

export class Router {
  private routes: Route[] = [];
  private middlewares: Middleware[] = [];

  // ルート登録（汎用）
  add(method: string, pattern: string | RegExp, handler: Handler): this {
    const regex =
      typeof pattern === 'string' ? this.pathToRegex(pattern) : pattern;

    this.routes.push({
      method,
      pattern: regex,
      handler,
    });

    return this;
  }

  // HTTPメソッド別のヘルパー
  get(pattern: string | RegExp, handler: Handler): this {
    return this.add('GET', pattern, handler);
  }

  post(pattern: string | RegExp, handler: Handler): this {
    return this.add('POST', pattern, handler);
  }

  put(pattern: string | RegExp, handler: Handler): this {
    return this.add('PUT', pattern, handler);
  }

  delete(pattern: string | RegExp, handler: Handler): this {
    return this.add('DELETE', pattern, handler);
  }

  patch(pattern: string | RegExp, handler: Handler): this {
    return this.add('PATCH', pattern, handler);
  }

  options(pattern: string | RegExp, handler: Handler): this {
    return this.add('OPTIONS', pattern, handler);
  }

  // すべてのメソッドにマッチ
  all(pattern: string | RegExp, handler: Handler): this {
    const methods = ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'];
    methods.forEach((method) => this.add(method, pattern, handler));
    return this;
  }

  // ミドルウェア追加
  use(middleware: Middleware): this {
    this.middlewares.push(middleware);
    return this;
  }

  // ルートグループ
  group(prefix: string, callback: (router: Router) => void): this {
    const groupRouter = new Router();
    callback(groupRouter);

    for (const route of groupRouter.routes) {
      const newPattern = this.addPrefix(route.pattern, prefix);
      this.routes.push({
        method: route.method,
        pattern: newPattern,
        handler: route.handler,
      });
    }

    return this;
  }

  // リクエスト処理
  async handle(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;
    const method = request.method;

    // ルートを探す
    for (const route of this.routes) {
      if (route.method !== method) {
        continue;
      }

      const match = path.match(route.pattern);

      if (match) {
        const params = match.groups || {};

        return this.runMiddlewares(request, () =>
          route.handler(request, params)
        );
      }
    }

    return new Response('Not Found', { status: 404 });
  }

  // パスパターンを正規表現に変換
  private pathToRegex(path: string): RegExp {
    const pattern = path
      .replace(/\//g, '\\/')
      .replace(/:(\w+)/g, '(?<$1>[^\\/]+)');

    return new RegExp(`^${pattern}$`);
  }

  // プレフィックス追加
  private addPrefix(pattern: RegExp, prefix: string): RegExp {
    const source = pattern.source;
    const cleanPrefix = prefix.replace(/^\/|\/$/g, '');
    const newSource = source.replace(/^\^\\\//, `^\\\/${cleanPrefix}\\/`);
    return new RegExp(newSource);
  }

  // ミドルウェア実行
  private async runMiddlewares(
    request: Request,
    finalHandler: () => Promise<Response>
  ): Promise<Response> {
    let index = 0;

    const next = async (): Promise<Response> => {
      if (index >= this.middlewares.length) {
        return finalHandler();
      }

      const middleware = this.middlewares[index++];
      return middleware(request, next);
    };

    return next();
  }
}
```

## Step 4-5: 実践例：完全なAPIサーバー

### src/index.ts

```typescript
import { Router } from './router';

const router = new Router();

// グローバルミドルウェア
router.use(async (request, next) => {
  const start = Date.now();
  const response = await next();
  const duration = Date.now() - start;

  const newHeaders = new Headers(response.headers);
  newHeaders.set('X-Response-Time', `${duration}ms`);

  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers: newHeaders,
  });
});

// CORS
router.use(async (request, next) => {
  if (request.method === 'OPTIONS') {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization',
      },
    });
  }

  const response = await next();
  const headers = new Headers(response.headers);
  headers.set('Access-Control-Allow-Origin', '*');

  return new Response(response.body, {
    status: response.status,
    headers,
  });
});

// ルート定義
router.get('/', (request, params) => {
  return new Response(
    JSON.stringify({
      message: 'Welcome to the API',
      version: '1.0.0',
      endpoints: {
        users: '/api/users',
        posts: '/api/posts',
      },
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
});

// APIルートグループ
router.group('/api', (api) => {
  // ユーザーAPI
  api.get('/users', (request, params) => {
    return new Response(
      JSON.stringify([
        { id: '1', name: 'Alice', email: 'alice@example.com' },
        { id: '2', name: 'Bob', email: 'bob@example.com' },
      ]),
      { headers: { 'Content-Type': 'application/json' } }
    );
  });

  api.get('/users/:id', (request, params) => {
    return new Response(
      JSON.stringify({
        id: params.id,
        name: `User ${params.id}`,
        email: `user${params.id}@example.com`,
      }),
      { headers: { 'Content-Type': 'application/json' } }
    );
  });

  api.post('/users', async (request, params) => {
    const body = await request.json();

    return new Response(
      JSON.stringify({
        id: Date.now().toString(),
        ...body,
        created: true,
      }),
      {
        status: 201,
        headers: { 'Content-Type': 'application/json' },
      }
    );
  });

  // 投稿API
  api.get('/posts', (request, params) => {
    return new Response(
      JSON.stringify([
        { id: '1', title: 'Post 1', content: 'Content 1' },
        { id: '2', title: 'Post 2', content: 'Content 2' },
      ]),
      { headers: { 'Content-Type': 'application/json' } }
    );
  });

  api.get('/posts/:id', (request, params) => {
    return new Response(
      JSON.stringify({
        id: params.id,
        title: `Post ${params.id}`,
        content: `Content of post ${params.id}`,
      }),
      { headers: { 'Content-Type': 'application/json' } }
    );
  });
});

export default {
  async fetch(request: Request): Promise<Response> {
    return router.handle(request);
  },
};
```

## 演習問題

### 問題1: エラーハンドリングの追加

Routerクラスに、ハンドラーがエラーをスローした場合に適切なエラーレスポンスを返す機能を追加してください。

### 問題2: パスの末尾スラッシュの正規化

`/users`と`/users/`を同じルートとして扱うように、Routerクラスを修正してください。

## まとめ

このステップで学んだこと：

✅ 再利用可能なRouterクラスの実装
✅ メソッド別のヘルパーメソッド
✅ ミドルウェアの統合
✅ ルートグループ機能
✅ 実践的なAPI設計

## 次のステップ

ルーティングの基礎をマスターしました！次のハンズオンでは、Workers KVを使ったデータ管理を学びます。

👉 [ハンズオン04: Workers KVでのデータ管理](../04-workers-kv/README.md)

## 参考

人気のルーティングライブラリ：
- [itty-router](https://github.com/kwhitley/itty-router) - 軽量でシンプル
- [Hono](https://hono.dev/) - 高速で機能豊富
- [worktop](https://github.com/lukeed/worktop) - TypeScript優先

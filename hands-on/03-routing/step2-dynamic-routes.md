# Step 2: 動的ルート

## このステップで学ぶこと

- パスパラメータの抽出
- 正規表現を使ったパターンマッチング
- 複数パラメータの処理
- パラメータのバリデーション

## Step 2-1: パスパラメータの基礎

静的ルート（`/users`）だけでは不十分です。`/users/123`のように、パスの一部を変数として扱いたい場合があります。

### 最初のアプローチ: split()を使う

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;
    const pathParts = path.split('/').filter(Boolean);

    // /users/123 → ['users', '123']
    // /posts/456/comments/789 → ['posts', '456', 'comments', '789']

    // GET /users/:id
    if (pathParts[0] === 'users' && pathParts.length === 2) {
      const userId = pathParts[1];
      return new Response(`User ID: ${userId}`);
    }

    // GET /posts/:id
    if (pathParts[0] === 'posts' && pathParts.length === 2) {
      const postId = pathParts[1];
      return new Response(`Post ID: ${postId}`);
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

### 動作確認

```bash
curl http://localhost:8787/users/123
# => User ID: 123

curl http://localhost:8787/posts/456
# => Post ID: 456
```

## Step 2-2: 正規表現を使ったパターンマッチング

より柔軟なマッチングには正規表現を使います。

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;

    // GET /users/:id
    const userPattern = /^\/users\/([a-zA-Z0-9]+)$/;
    const userMatch = path.match(userPattern);

    if (userMatch) {
      const userId = userMatch[1];
      return handleUser(userId);
    }

    // GET /posts/:postId/comments/:commentId
    const commentPattern = /^\/posts\/([a-zA-Z0-9]+)\/comments\/([a-zA-Z0-9]+)$/;
    const commentMatch = path.match(commentPattern);

    if (commentMatch) {
      const postId = commentMatch[1];
      const commentId = commentMatch[2];
      return handleComment(postId, commentId);
    }

    // GET /api/v1/users/:id
    const apiUserPattern = /^\/api\/v1\/users\/([a-zA-Z0-9]+)$/;
    const apiUserMatch = path.match(apiUserPattern);

    if (apiUserMatch) {
      const userId = apiUserMatch[1];
      return handleApiUser(userId);
    }

    return new Response('Not Found', { status: 404 });
  },
};

function handleUser(userId: string): Response {
  return new Response(
    JSON.stringify({
      userId,
      name: `User ${userId}`,
      email: `user${userId}@example.com`,
    }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
}

function handleComment(postId: string, commentId: string): Response {
  return new Response(
    JSON.stringify({
      postId,
      commentId,
      text: `Comment ${commentId} on Post ${postId}`,
    }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
}

function handleApiUser(userId: string): Response {
  return new Response(
    JSON.stringify({
      version: 'v1',
      userId,
      data: { name: `API User ${userId}` },
    }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
}
```

### 動作確認

```bash
curl http://localhost:8787/users/alice
# => {"userId":"alice","name":"User alice","email":"useralice@example.com"}

curl http://localhost:8787/posts/123/comments/456
# => {"postId":"123","commentId":"456","text":"Comment 456 on Post 123"}

curl http://localhost:8787/api/v1/users/bob
# => {"version":"v1","userId":"bob","data":{"name":"API User bob"}}
```

## Step 2-3: パラメータ抽出の汎用関数

パターンマッチングを汎用化します。

```typescript
interface RouteMatch {
  params: Record<string, string>;
  handler: RouteHandler;
}

type RouteHandler = (
  request: Request,
  params: Record<string, string>
) => Response | Promise<Response>;

interface Route {
  pattern: RegExp;
  handler: RouteHandler;
}

// ルート定義
const routes: Route[] = [
  {
    pattern: /^\/users\/(?<id>[a-zA-Z0-9]+)$/,
    handler: handleUser,
  },
  {
    pattern: /^\/posts\/(?<postId>[a-zA-Z0-9]+)$/,
    handler: handlePost,
  },
  {
    pattern: /^\/posts\/(?<postId>[a-zA-Z0-9]+)\/comments\/(?<commentId>[a-zA-Z0-9]+)$/,
    handler: handleComment,
  },
];

export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;

    // ルートをマッチング
    for (const route of routes) {
      const match = path.match(route.pattern);

      if (match && match.groups) {
        return route.handler(request, match.groups);
      }
    }

    return new Response('Not Found', { status: 404 });
  },
};

// ハンドラー関数
function handleUser(
  request: Request,
  params: Record<string, string>
): Response {
  const { id } = params;

  return new Response(
    JSON.stringify({
      userId: id,
      name: `User ${id}`,
      email: `user${id}@example.com`,
    }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
}

function handlePost(
  request: Request,
  params: Record<string, string>
): Response {
  const { postId } = params;

  return new Response(
    JSON.stringify({
      postId,
      title: `Post ${postId}`,
      content: `Content of post ${postId}`,
    }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
}

function handleComment(
  request: Request,
  params: Record<string, string>
): Response {
  const { postId, commentId } = params;

  return new Response(
    JSON.stringify({
      postId,
      commentId,
      text: `Comment ${commentId} on Post ${postId}`,
    }),
    {
      headers: { 'Content-Type': 'application/json' },
    }
  );
}
```

### 名前付きキャプチャグループの説明

正規表現の`(?<name>pattern)`構文を使うと、キャプチャした値に名前を付けられます：

```typescript
const pattern = /^\/users\/(?<id>[a-zA-Z0-9]+)$/;
const match = '/users/123'.match(pattern);

if (match && match.groups) {
  console.log(match.groups.id); // => "123"
}
```

## Step 2-4: HTTPメソッドも考慮する

パスだけでなく、HTTPメソッドも含めてマッチングします。

```typescript
interface Route {
  method: string;
  pattern: RegExp;
  handler: RouteHandler;
}

const routes: Route[] = [
  {
    method: 'GET',
    pattern: /^\/users$/,
    handler: listUsers,
  },
  {
    method: 'POST',
    pattern: /^\/users$/,
    handler: createUser,
  },
  {
    method: 'GET',
    pattern: /^\/users\/(?<id>[a-zA-Z0-9]+)$/,
    handler: getUser,
  },
  {
    method: 'PUT',
    pattern: /^\/users\/(?<id>[a-zA-Z0-9]+)$/,
    handler: updateUser,
  },
  {
    method: 'DELETE',
    pattern: /^\/users\/(?<id>[a-zA-Z0-9]+)$/,
    handler: deleteUser,
  },
];

export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;
    const method = request.method;

    // ルートをマッチング
    for (const route of routes) {
      if (route.method !== method) {
        continue;
      }

      const match = path.match(route.pattern);

      if (match) {
        const params = match.groups || {};
        return route.handler(request, params);
      }
    }

    return new Response('Not Found', { status: 404 });
  },
};

// ハンドラー関数
function listUsers(
  request: Request,
  params: Record<string, string>
): Response {
  return new Response(
    JSON.stringify([
      { id: '1', name: 'Alice' },
      { id: '2', name: 'Bob' },
    ]),
    { headers: { 'Content-Type': 'application/json' } }
  );
}

async function createUser(
  request: Request,
  params: Record<string, string>
): Promise<Response> {
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
}

function getUser(
  request: Request,
  params: Record<string, string>
): Response {
  const { id } = params;

  return new Response(
    JSON.stringify({
      id,
      name: `User ${id}`,
      email: `user${id}@example.com`,
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
}

async function updateUser(
  request: Request,
  params: Record<string, string>
): Promise<Response> {
  const { id } = params;
  const body = await request.json();

  return new Response(
    JSON.stringify({
      id,
      ...body,
      updated: true,
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
}

function deleteUser(
  request: Request,
  params: Record<string, string>
): Response {
  const { id } = params;

  return new Response(
    JSON.stringify({
      message: `User ${id} deleted`,
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
}
```

### 動作確認

```bash
# GET /users - 一覧取得
curl http://localhost:8787/users

# POST /users - 作成
curl -X POST http://localhost:8787/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Charlie"}'

# GET /users/123 - 取得
curl http://localhost:8787/users/123

# PUT /users/123 - 更新
curl -X PUT http://localhost:8787/users/123 \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice Updated"}'

# DELETE /users/123 - 削除
curl -X DELETE http://localhost:8787/users/123
```

## Step 2-5: パラメータのバリデーション

パラメータが正しい形式かチェックします。

```typescript
// バリデーション関数
function validateUserId(id: string): boolean {
  // 数値のみ許可
  return /^\d+$/.test(id);
}

function validateUUID(id: string): boolean {
  // UUID形式
  return /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(
    id
  );
}

// ハンドラー内でバリデーション
function getUser(
  request: Request,
  params: Record<string, string>
): Response {
  const { id } = params;

  // バリデーション
  if (!validateUserId(id)) {
    return new Response(
      JSON.stringify({
        error: 'Invalid user ID format',
      }),
      {
        status: 400,
        headers: { 'Content-Type': 'application/json' },
      }
    );
  }

  return new Response(
    JSON.stringify({
      id,
      name: `User ${id}`,
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
}
```

## 演習問題

### 問題1: ネストされたリソース

以下のルートを実装してください：

- `GET /categories/:categoryId/products/:productId`
- カテゴリIDと商品IDを含むJSONを返す

<details>
<summary>解答例</summary>

```typescript
const routes: Route[] = [
  // ...他のルート
  {
    method: 'GET',
    pattern: /^\/categories\/(?<categoryId>[a-zA-Z0-9]+)\/products\/(?<productId>[a-zA-Z0-9]+)$/,
    handler: getProduct,
  },
];

function getProduct(
  request: Request,
  params: Record<string, string>
): Response {
  const { categoryId, productId } = params;

  return new Response(
    JSON.stringify({
      categoryId,
      productId,
      name: `Product ${productId} in Category ${categoryId}`,
      price: 1000,
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
}
```
</details>

### 問題2: パラメータのバリデーション

商品IDが数値のみであることをバリデーションする機能を追加してください。

<details>
<summary>解答例</summary>

```typescript
function getProduct(
  request: Request,
  params: Record<string, string>
): Response {
  const { categoryId, productId } = params;

  // バリデーション
  if (!/^\d+$/.test(productId)) {
    return new Response(
      JSON.stringify({
        error: 'Product ID must be numeric',
      }),
      {
        status: 400,
        headers: { 'Content-Type': 'application/json' },
      }
    );
  }

  return new Response(
    JSON.stringify({
      categoryId,
      productId,
      name: `Product ${productId}`,
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
}
```
</details>

## まとめ

このステップで学んだこと：

✅ パスパラメータの抽出
✅ 正規表現を使ったパターンマッチング
✅ 名前付きキャプチャグループ
✅ HTTPメソッドとパスの組み合わせ
✅ パラメータのバリデーション

## 次のステップ

次はミドルウェアパターンを学びます。認証やログなど、共通処理を効率的に実装する方法を学びましょう。

👉 [Step 3: ミドルウェアパターン](./step3-middleware.md)

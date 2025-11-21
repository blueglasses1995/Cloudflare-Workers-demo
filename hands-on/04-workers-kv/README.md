# ハンズオン04: Workers KVでのデータ管理

## 学習目標

- Workers KVの基本操作
- KVを使ったデータの永続化
- キャッシュ戦略の実装
- セッション管理
- 実践的なアプリケーション開発

## 事前準備

```bash
mkdir kv-demo-worker
cd kv-demo-worker
wrangler init
```

## Step 1: KVネームスペースの作成

### KVネームスペースを作成

```bash
# 本番環境用
wrangler kv:namespace create "MY_KV"

# プレビュー環境用（開発用）
wrangler kv:namespace create "MY_KV" --preview
```

出力例：
```
✨ Success!
Add the following to your wrangler.toml:
[[kv_namespaces]]
binding = "MY_KV"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
preview_id = "yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy"
```

### wrangler.tomlに追加

```toml
name = "kv-demo-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[[kv_namespaces]]
binding = "MY_KV"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
preview_id = "yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy"
```

## Step 2: 基本的なKV操作

### 型定義

```typescript
export interface Env {
  MY_KV: KVNamespace;
}
```

### CRUD操作の実装

```typescript
export interface Env {
  MY_KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;

    // POST /set - データを保存
    if (path === '/set' && request.method === 'POST') {
      const { key, value } = await request.json();
      await env.MY_KV.put(key, value);

      return new Response(
        JSON.stringify({ message: 'Data saved', key }),
        { headers: { 'Content-Type': 'application/json' } }
      );
    }

    // GET /get/:key - データを取得
    if (path.startsWith('/get/')) {
      const key = path.split('/')[2];
      const value = await env.MY_KV.get(key);

      if (value === null) {
        return new Response('Not Found', { status: 404 });
      }

      return new Response(
        JSON.stringify({ key, value }),
        { headers: { 'Content-Type': 'application/json' } }
      );
    }

    // DELETE /delete/:key - データを削除
    if (path.startsWith('/delete/')) {
      const key = path.split('/')[2];
      await env.MY_KV.delete(key);

      return new Response(
        JSON.stringify({ message: 'Data deleted', key }),
        { headers: { 'Content-Type': 'application/json' } }
      );
    }

    // GET /list - キー一覧を取得
    if (path === '/list') {
      const list = await env.MY_KV.list();
      const keys = list.keys.map(k => k.name);

      return new Response(
        JSON.stringify({ keys }),
        { headers: { 'Content-Type': 'application/json' } }
      );
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

### テスト

```bash
# ローカル開発サーバー起動
wrangler dev

# データを保存
curl -X POST http://localhost:8787/set \
  -H "Content-Type: application/json" \
  -d '{"key": "user:1", "value": "Alice"}'

# データを取得
curl http://localhost:8787/get/user:1

# キー一覧を取得
curl http://localhost:8787/list

# データを削除
curl -X DELETE http://localhost:8787/delete/user:1
```

## Step 3: JSONデータの保存と取得

### JSON操作の実装

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  createdAt: string;
}

export interface Env {
  MY_KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // POST /users - ユーザー作成
    if (url.pathname === '/users' && request.method === 'POST') {
      const userData = await request.json();
      const user: User = {
        id: Date.now(),
        name: userData.name,
        email: userData.email,
        createdAt: new Date().toISOString(),
      };

      // JSONとして保存
      await env.MY_KV.put(
        `user:${user.id}`,
        JSON.stringify(user)
      );

      return new Response(JSON.stringify(user), {
        status: 201,
        headers: { 'Content-Type': 'application/json' },
      });
    }

    // GET /users/:id - ユーザー取得
    if (url.pathname.startsWith('/users/') && request.method === 'GET') {
      const userId = url.pathname.split('/')[2];
      const key = `user:${userId}`;

      // JSONとして取得
      const userData = await env.MY_KV.get(key, { type: 'json' });

      if (!userData) {
        return new Response('User not found', { status: 404 });
      }

      return new Response(JSON.stringify(userData), {
        headers: { 'Content-Type': 'application/json' },
      });
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

## Step 4: 有効期限付きデータ

### TTL設定

```typescript
export interface Env {
  MY_KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // POST /cache - キャッシュデータを保存（1時間）
    if (url.pathname === '/cache' && request.method === 'POST') {
      const { key, value } = await request.json();

      // 1時間（3600秒）後に自動削除
      await env.MY_KV.put(key, value, {
        expirationTtl: 3600,
      });

      return new Response(
        JSON.stringify({ message: 'Cached for 1 hour', key }),
        { headers: { 'Content-Type': 'application/json' } }
      );
    }

    // POST /temp - 一時データを保存（1分）
    if (url.pathname === '/temp' && request.method === 'POST') {
      const { key, value } = await request.json();

      // 1分（60秒）後に自動削除
      await env.MY_KV.put(key, value, {
        expirationTtl: 60,
      });

      return new Response(
        JSON.stringify({ message: 'Temporary data (1 minute)', key }),
        { headers: { 'Content-Type': 'application/json' } }
      );
    }

    // 特定の日時に削除
    if (url.pathname === '/expire-at' && request.method === 'POST') {
      const { key, value, expireAt } = await request.json();

      // Unix timestamp（秒）で指定
      const expirationTimestamp = Math.floor(new Date(expireAt).getTime() / 1000);

      await env.MY_KV.put(key, value, {
        expiration: expirationTimestamp,
      });

      return new Response(
        JSON.stringify({ message: `Expires at ${expireAt}`, key }),
        { headers: { 'Content-Type': 'application/json' } }
      );
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

## Step 5: メタデータの活用

### メタデータ付きデータ

```typescript
interface UserMetadata {
  createdBy: string;
  updatedAt: string;
  tags: string[];
}

export interface Env {
  MY_KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // POST /users - メタデータ付きで保存
    if (url.pathname === '/users' && request.method === 'POST') {
      const userData = await request.json();
      const userId = Date.now();

      const metadata: UserMetadata = {
        createdBy: 'admin',
        updatedAt: new Date().toISOString(),
        tags: userData.tags || [],
      };

      await env.MY_KV.put(
        `user:${userId}`,
        JSON.stringify(userData),
        { metadata }
      );

      return new Response(
        JSON.stringify({ id: userId, ...userData }),
        {
          status: 201,
          headers: { 'Content-Type': 'application/json' },
        }
      );
    }

    // GET /users/:id - メタデータ付きで取得
    if (url.pathname.startsWith('/users/') && request.method === 'GET') {
      const userId = url.pathname.split('/')[2];
      const key = `user:${userId}`;

      const { value, metadata } = await env.MY_KV.getWithMetadata(key, {
        type: 'json',
      });

      if (!value) {
        return new Response('User not found', { status: 404 });
      }

      return new Response(
        JSON.stringify({ data: value, metadata }),
        { headers: { 'Content-Type': 'application/json' } }
      );
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

## Step 6: キャッシュパターンの実装

### APIレスポンスのキャッシュ

```typescript
export interface Env {
  MY_KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === '/weather') {
      const city = url.searchParams.get('city') || 'Tokyo';
      const cacheKey = `weather:${city}`;

      // キャッシュを確認
      const cached = await env.MY_KV.get(cacheKey, { type: 'json' });

      if (cached) {
        return new Response(
          JSON.stringify({
            data: cached,
            source: 'cache',
          }),
          {
            headers: {
              'Content-Type': 'application/json',
              'X-Cache': 'HIT',
            },
          }
        );
      }

      // キャッシュミス: 外部APIから取得
      const apiResponse = await fetch(
        `https://api.weatherapi.com/v1/current.json?key=YOUR_API_KEY&q=${city}`
      );
      const data = await apiResponse.json();

      // 10分間キャッシュ
      await env.MY_KV.put(cacheKey, JSON.stringify(data), {
        expirationTtl: 600,
      });

      return new Response(
        JSON.stringify({
          data,
          source: 'api',
        }),
        {
          headers: {
            'Content-Type': 'application/json',
            'X-Cache': 'MISS',
          },
        }
      );
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

## Step 7: セッション管理

### セッションストア実装

```typescript
interface Session {
  userId: string;
  username: string;
  createdAt: string;
  expiresAt: string;
}

export interface Env {
  MY_KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // POST /login - ログイン（セッション作成）
    if (url.pathname === '/login' && request.method === 'POST') {
      const { username, password } = await request.json();

      // 認証処理（簡略化）
      if (password !== 'password123') {
        return new Response('Invalid credentials', { status: 401 });
      }

      // セッションID生成
      const sessionId = crypto.randomUUID();
      const now = new Date();
      const expiresAt = new Date(now.getTime() + 24 * 60 * 60 * 1000); // 24時間

      const session: Session = {
        userId: username,
        username,
        createdAt: now.toISOString(),
        expiresAt: expiresAt.toISOString(),
      };

      // セッションを保存（24時間）
      await env.MY_KV.put(
        `session:${sessionId}`,
        JSON.stringify(session),
        { expirationTtl: 86400 }
      );

      // Cookieを設定
      return new Response(
        JSON.stringify({ message: 'Login successful', sessionId }),
        {
          headers: {
            'Content-Type': 'application/json',
            'Set-Cookie': `session=${sessionId}; Path=/; Max-Age=86400; HttpOnly; Secure; SameSite=Strict`,
          },
        }
      );
    }

    // GET /profile - プロフィール取得（認証必要）
    if (url.pathname === '/profile' && request.method === 'GET') {
      // Cookieからセッション ID を取得
      const cookies = request.headers.get('Cookie') || '';
      const sessionId = cookies
        .split(';')
        .find(c => c.trim().startsWith('session='))
        ?.split('=')[1];

      if (!sessionId) {
        return new Response('Unauthorized', { status: 401 });
      }

      // セッションを取得
      const sessionData = await env.MY_KV.get(`session:${sessionId}`, {
        type: 'json',
      });

      if (!sessionData) {
        return new Response('Session expired', { status: 401 });
      }

      const session = sessionData as Session;

      return new Response(
        JSON.stringify({
          username: session.username,
          userId: session.userId,
        }),
        { headers: { 'Content-Type': 'application/json' } }
      );
    }

    // POST /logout - ログアウト
    if (url.pathname === '/logout' && request.method === 'POST') {
      const cookies = request.headers.get('Cookie') || '';
      const sessionId = cookies
        .split(';')
        .find(c => c.trim().startsWith('session='))
        ?.split('=')[1];

      if (sessionId) {
        // セッションを削除
        await env.MY_KV.delete(`session:${sessionId}`);
      }

      return new Response(
        JSON.stringify({ message: 'Logged out' }),
        {
          headers: {
            'Content-Type': 'application/json',
            'Set-Cookie': 'session=; Path=/; Max-Age=0',
          },
        }
      );
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

## Step 8: プリフィックスを使ったリスト取得

### プリフィックスベースのクエリ

```typescript
export interface Env {
  MY_KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // GET /users - ユーザー一覧
    if (url.pathname === '/users') {
      const list = await env.MY_KV.list({ prefix: 'user:' });

      // すべてのユーザーデータを取得
      const users = await Promise.all(
        list.keys.map(async key => {
          const data = await env.MY_KV.get(key.name, { type: 'json' });
          return data;
        })
      );

      return new Response(JSON.stringify(users), {
        headers: { 'Content-Type': 'application/json' },
      });
    }

    // GET /posts - 投稿一覧（ページネーション）
    if (url.pathname === '/posts') {
      const cursor = url.searchParams.get('cursor') || undefined;
      const limit = 10;

      const list = await env.MY_KV.list({
        prefix: 'post:',
        limit,
        cursor,
      });

      const posts = await Promise.all(
        list.keys.map(async key => {
          const data = await env.MY_KV.get(key.name, { type: 'json' });
          return data;
        })
      );

      return new Response(
        JSON.stringify({
          posts,
          cursor: list.cursor,
          hasMore: !list.list_complete,
        }),
        { headers: { 'Content-Type': 'application/json' } }
      );
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

## Step 9: Wrangler CLIでのKV操作

### コマンドラインからのKV操作

```bash
# キーと値を設定
wrangler kv:key put --binding=MY_KV "key1" "value1"

# 値を取得
wrangler kv:key get --binding=MY_KV "key1"

# キーを削除
wrangler kv:key delete --binding=MY_KV "key1"

# キー一覧を表示
wrangler kv:key list --binding=MY_KV

# ファイルから一括登録
wrangler kv:bulk put --binding=MY_KV data.json
```

### data.jsonの例

```json
[
  { "key": "user:1", "value": "{\"name\":\"Alice\"}" },
  { "key": "user:2", "value": "{\"name\":\"Bob\"}" },
  { "key": "user:3", "value": "{\"name\":\"Charlie\"}" }
]
```

## 演習問題

### 問題1: ページビューカウンター

各ページのビュー数をKVに保存し、取得できるAPIを作成してください。

<details>
<summary>解答例</summary>

```typescript
export interface Env {
  MY_KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const page = url.searchParams.get('page') || 'home';
    const key = `pageview:${page}`;

    // 現在のカウントを取得
    const currentCount = await env.MY_KV.get(key);
    const count = currentCount ? parseInt(currentCount) : 0;

    // インクリメント
    const newCount = count + 1;
    await env.MY_KV.put(key, newCount.toString());

    return new Response(
      JSON.stringify({ page, views: newCount }),
      { headers: { 'Content-Type': 'application/json' } }
    );
  },
};
```
</details>

### 問題2: 簡易投票システム

投票内容を保存し、結果を集計できるシステムを作成してください。

### 問題3: レート制限

IPアドレスごとに1分間のリクエスト数を制限する機能を実装してください。

## まとめ

このハンズオンで学んだこと：

✅ KVネームスペースの作成と設定
✅ 基本的なCRUD操作
✅ JSONデータの保存と取得
✅ TTL（有効期限）の設定
✅ メタデータの活用
✅ キャッシュパターン
✅ セッション管理
✅ プリフィックスを使ったクエリ

## 次のステップ

- [ハンズオン05: REST API開発](../05-rest-api/README.md)

## 参考リソース

- [Workers KV Documentation](https://developers.cloudflare.com/workers/runtime-apis/kv/)

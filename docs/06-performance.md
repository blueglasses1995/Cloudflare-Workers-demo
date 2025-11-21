# 06. パフォーマンス最適化

## キャッシング戦略

### Cache API

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const cache = caches.default;
    const cacheKey = new Request(request.url, request);

    let response = await cache.match(cacheKey);

    if (!response) {
      response = await fetch(request);

      // 成功レスポンスのみキャッシュ
      if (response.ok) {
        const headers = new Headers(response.headers);
        headers.set('Cache-Control', 'public, max-age=3600');

        response = new Response(response.body, {
          status: response.status,
          statusText: response.statusText,
          headers,
        });

        await cache.put(cacheKey, response.clone());
      }
    }

    return response;
  },
};
```

### KVキャッシング

```typescript
export interface Env {
  CACHE: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const cacheKey = url.pathname;

    // KVからキャッシュ取得
    const cached = await env.CACHE.get(cacheKey, { type: 'json' });

    if (cached) {
      return new Response(JSON.stringify(cached), {
        headers: {
          'Content-Type': 'application/json',
          'X-Cache': 'HIT',
        },
      });
    }

    // データ取得
    const data = await fetchData();

    // KVにキャッシュ（1時間）
    await env.CACHE.put(cacheKey, JSON.stringify(data), {
      expirationTtl: 3600,
    });

    return new Response(JSON.stringify(data), {
      headers: {
        'Content-Type': 'application/json',
        'X-Cache': 'MISS',
      },
    });
  },
};
```

## 並列処理

### Promise.all

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    // 複数のリクエストを並列実行
    const [userData, postsData, commentsData] = await Promise.all([
      fetch('https://api.example.com/user'),
      fetch('https://api.example.com/posts'),
      fetch('https://api.example.com/comments'),
    ]);

    const [user, posts, comments] = await Promise.all([
      userData.json(),
      postsData.json(),
      commentsData.json(),
    ]);

    return new Response(JSON.stringify({ user, posts, comments }));
  },
};
```

## ストリーミング

### レスポンスのストリーミング

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const stream = new ReadableStream({
      async start(controller) {
        for (let i = 0; i < 10; i++) {
          const data = await fetchDataChunk(i);
          controller.enqueue(new TextEncoder().encode(JSON.stringify(data) + '\n'));
        }
        controller.close();
      },
    });

    return new Response(stream, {
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
```

## バンドルサイズの最適化

### Tree Shaking

```typescript
// ❌ 全体をインポート
import _ from 'lodash';

// ✅ 必要な関数のみインポート
import { map } from 'lodash-es';
```

### 動的インポート

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === '/heavy-feature') {
      // 必要な時だけロード
      const { heavyFunction } = await import('./heavy-module');
      return heavyFunction(request);
    }

    return new Response('OK');
  },
};
```

## CPU時間の最適化

### 効率的なアルゴリズム

```typescript
// ❌ 非効率
function findUser(users: User[], id: string) {
  for (const user of users) {
    if (user.id === id) return user;
  }
}

// ✅ 効率的
const userMap = new Map(users.map(u => [u.id, u]));
const user = userMap.get(id);
```

### 早期リターン

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    // 認証チェックを先に実行
    if (!isAuthenticated(request)) {
      return new Response('Unauthorized', { status: 401 });
    }

    // 重い処理は後で
    const data = await heavyOperation();

    return new Response(JSON.stringify(data));
  },
};
```

## メモリ最適化

### データの適切な破棄

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    let largeData = await fetchLargeData();

    // 必要な部分だけ抽出
    const result = processData(largeData);

    // 不要になったデータを明示的に破棄
    largeData = null;

    return new Response(JSON.stringify(result));
  },
};
```

## ベストプラクティス

1. **適切なキャッシング**: TTLを設定
2. **並列処理**: 可能な限り並列化
3. **ストリーミング**: 大きなデータはストリーミング
4. **バンドル最適化**: 不要なコードを削除
5. **早期リターン**: 無駄な処理を避ける
6. **メモリ管理**: 大きなオブジェクトは適切に破棄

## パフォーマンス測定

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const start = Date.now();

    const response = await handleRequest(request);

    const duration = Date.now() - start;

    response.headers.set('X-Response-Time', `${duration}ms`);

    console.log(`Request processed in ${duration}ms`);

    return response;
  },
};
```

---

次: [07. セキュリティベストプラクティス](./07-security.md)

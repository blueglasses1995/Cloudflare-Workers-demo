# Step 3: キャッシング戦略

## Workers KVでのキャッシング

```typescript
interface Env {
  API_CACHE: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const userId = url.searchParams.get('userId') || '1';
    const cacheKey = `user:${userId}`;

    // キャッシュチェック
    const cached = await env.API_CACHE.get(cacheKey, { type: 'json' });
    if (cached) {
      return new Response(JSON.stringify(cached), {
        headers: {
          'Content-Type': 'application/json',
          'X-Cache': 'HIT',
        },
      });
    }

    // API呼び出し
    const apiResponse = await fetch(
      `https://jsonplaceholder.typicode.com/users/${userId}`
    );
    const data = await apiResponse.json();

    // キャッシュに保存（1時間）
    await env.API_CACHE.put(cacheKey, JSON.stringify(data), {
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

## Cache APIの活用

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const cache = caches.default;
    const cacheKey = new Request(
      'https://jsonplaceholder.typicode.com/users/1',
      request
    );

    let response = await cache.match(cacheKey);

    if (!response) {
      response = await fetch(cacheKey);

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

👉 [Step 4: タイムアウトと並列処理](./step4-timeout-parallel.md)

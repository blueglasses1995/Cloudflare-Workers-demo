# Step 4: タイムアウトと並列処理

## AbortControllerによるタイムアウト

```typescript
async function fetchWithTimeout(
  url: string,
  options: RequestInit = {},
  timeout: number = 5000
): Promise<Response> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(url, {
      ...options,
      signal: controller.signal,
    });
    return response;
  } finally {
    clearTimeout(timeoutId);
  }
}

export default {
  async fetch(request: Request): Promise<Response> {
    try {
      const response = await fetchWithTimeout(
        'https://jsonplaceholder.typicode.com/users/1',
        {},
        3000 // 3秒でタイムアウト
      );

      const data = await response.json();
      return new Response(JSON.stringify(data), {
        headers: { 'Content-Type': 'application/json' },
      });
    } catch (error) {
      return new Response(
        JSON.stringify({ error: 'Request timeout' }),
        {
          status: 504,
          headers: { 'Content-Type': 'application/json' },
        }
      );
    }
  },
};
```

## 複数APIの並列呼び出し

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const [user, posts, todos] = await Promise.all([
      fetch('https://jsonplaceholder.typicode.com/users/1').then((r) => r.json()),
      fetch('https://jsonplaceholder.typicode.com/posts?userId=1').then((r) => r.json()),
      fetch('https://jsonplaceholder.typicode.com/todos?userId=1').then((r) => r.json()),
    ]);

    return new Response(
      JSON.stringify({
        user,
        posts,
        todos,
      }),
      { headers: { 'Content-Type': 'application/json' } }
    );
  },
};
```

## Promise.raceでフォールバック

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const primaryAPI = fetch('https://api.primary.com/data');
    const fallbackAPI = fetch('https://api.fallback.com/data');

    const response = await Promise.race([primaryAPI, fallbackAPI]);
    const data = await response.json();

    return new Response(JSON.stringify(data), {
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
```

👉 [Step 5: 実践プロジェクト](./step5-practical-project.md)

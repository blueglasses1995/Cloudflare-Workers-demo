# Step 1: 基本的な外部API呼び出し

## 基本的なGETリクエスト

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    // 外部APIを呼び出し
    const apiResponse = await fetch('https://jsonplaceholder.typicode.com/users/1');
    const data = await apiResponse.json();

    return new Response(JSON.stringify(data, null, 2), {
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
```

## クエリパラメータの追加

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const userId = url.searchParams.get('userId') || '1';

    const apiURL = `https://jsonplaceholder.typicode.com/users/${userId}`;
    const apiResponse = await fetch(apiURL);
    const data = await apiResponse.json();

    return new Response(JSON.stringify(data), {
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
```

## POSTリクエスト

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const body = {
      title: 'New Post',
      body: 'This is the content',
      userId: 1,
    };

    const apiResponse = await fetch('https://jsonplaceholder.typicode.com/posts', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(body),
    });

    const data = await apiResponse.json();

    return new Response(JSON.stringify(data), {
      status: 201,
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
```

## カスタムヘッダーの設定

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const apiResponse = await fetch('https://api.example.com/data', {
      headers: {
        'Authorization': `Bearer ${env.API_KEY}`,
        'User-Agent': 'Cloudflare-Worker',
        'Accept': 'application/json',
      },
    });

    const data = await apiResponse.json();

    return new Response(JSON.stringify(data), {
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
```

👉 [Step 2: エラーハンドリングとリトライ](./step2-error-handling.md)

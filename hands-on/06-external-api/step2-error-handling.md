# Step 2: エラーハンドリングとリトライ

## HTTPステータスコードの処理

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const apiResponse = await fetch('https://jsonplaceholder.typicode.com/users/1');

    if (!apiResponse.ok) {
      return new Response(
        JSON.stringify({
          error: 'API request failed',
          status: apiResponse.status,
          statusText: apiResponse.statusText,
        }),
        {
          status: apiResponse.status,
          headers: { 'Content-Type': 'application/json' },
        }
      );
    }

    const data = await apiResponse.json();
    return new Response(JSON.stringify(data), {
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
```

## リトライロジック

```typescript
async function fetchWithRetry(
  url: string,
  options: RequestInit = {},
  maxRetries: number = 3
): Promise<Response> {
  let lastError: Error;

  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url, options);

      if (response.ok) {
        return response;
      }

      // サーバーエラー（5xx）の場合のみリトライ
      if (response.status >= 500) {
        throw new Error(`Server error: ${response.status}`);
      }

      return response;
    } catch (error) {
      lastError = error as Error;
      console.log(`Retry ${i + 1}/${maxRetries}:`, lastError.message);

      // 最後のリトライでなければ待機
      if (i < maxRetries - 1) {
        await new Promise((resolve) => setTimeout(resolve, 1000 * (i + 1)));
      }
    }
  }

  throw lastError!;
}
```

## エクスポネンシャルバックオフ

```typescript
async function fetchWithExponentialBackoff(
  url: string,
  options: RequestInit = {},
  maxRetries: number = 3
): Promise<Response> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url, options);

      if (response.ok || response.status < 500) {
        return response;
      }

      if (i < maxRetries - 1) {
        // エクスポネンシャルバックオフ: 2^i * 1000ms
        const delay = Math.pow(2, i) * 1000;
        await new Promise((resolve) => setTimeout(resolve, delay));
      }
    } catch (error) {
      if (i === maxRetries - 1) throw error;

      const delay = Math.pow(2, i) * 1000;
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  throw new Error('Max retries exceeded');
}
```

👉 [Step 3: キャッシング戦略](./step3-caching.md)

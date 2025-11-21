# Step 1: API Key認証

```typescript
interface Env {
  API_KEY: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const apiKey = request.headers.get('X-API-Key');
    
    if (!apiKey || apiKey !== env.API_KEY) {
      return new Response('Unauthorized', { status: 401 });
    }
    
    return new Response('Authorized');
  },
};
```

👉 [Step 2: JWT認証](./step2-jwt.md)

# Step 2: JWT認証

```typescript
import { verify } from '@tsndr/cloudflare-worker-jwt';

interface Env {
  JWT_SECRET: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const authHeader = request.headers.get('Authorization');
    if (!authHeader?.startsWith('Bearer ')) {
      return new Response('Missing token', { status: 401 });
    }

    const token = authHeader.substring(7);
    const isValid = await verify(token, env.JWT_SECRET);

    if (!isValid) {
      return new Response('Invalid token', { status: 401 });
    }

    return new Response('Authorized');
  },
};
```

👉 [Step 3: セッション管理](./step3-sessions.md)

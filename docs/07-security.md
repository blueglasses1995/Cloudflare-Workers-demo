# 07. セキュリティベストプラクティス

## 認証と認可

### API Key認証

```typescript
export interface Env {
  API_KEY: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const apiKey = request.headers.get('X-API-Key');

    if (apiKey !== env.API_KEY) {
      return new Response('Unauthorized', {
        status: 401,
        headers: {
          'WWW-Authenticate': 'API-Key',
        },
      });
    }

    return new Response('Authorized');
  },
};
```

### JWT認証

```typescript
import { verify } from '@tsndr/cloudflare-worker-jwt';

export interface Env {
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

## セキュリティヘッダー

### 推奨ヘッダー設定

```typescript
function addSecurityHeaders(response: Response): Response {
  const newHeaders = new Headers(response.headers);

  // XSS保護
  newHeaders.set('X-Content-Type-Options', 'nosniff');
  newHeaders.set('X-Frame-Options', 'DENY');
  newHeaders.set('X-XSS-Protection', '1; mode=block');

  // HTTPS強制
  newHeaders.set(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains; preload'
  );

  // CSP
  newHeaders.set(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' 'unsafe-inline'"
  );

  // Referrer制御
  newHeaders.set('Referrer-Policy', 'strict-origin-when-cross-origin');

  // Permissions Policy
  newHeaders.set(
    'Permissions-Policy',
    'geolocation=(), microphone=(), camera=()'
  );

  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers: newHeaders,
  });
}

export default {
  async fetch(request: Request): Promise<Response> {
    const response = await handleRequest(request);
    return addSecurityHeaders(response);
  },
};
```

## 入力検証

### リクエストボディの検証

```typescript
interface UserInput {
  email: string;
  password: string;
}

function validateEmail(email: string): boolean {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return regex.test(email);
}

function validatePassword(password: string): boolean {
  return password.length >= 8;
}

export default {
  async fetch(request: Request): Promise<Response> {
    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    let data: UserInput;

    try {
      data = await request.json();
    } catch {
      return new Response('Invalid JSON', { status: 400 });
    }

    // バリデーション
    if (!validateEmail(data.email)) {
      return new Response('Invalid email', { status: 400 });
    }

    if (!validatePassword(data.password)) {
      return new Response('Password must be at least 8 characters', {
        status: 400,
      });
    }

    return new Response('Valid input');
  },
};
```

## レート制限

### IPベースのレート制限

```typescript
export interface Env {
  RATE_LIMIT: KVNamespace;
}

const RATE_LIMIT_WINDOW = 60; // 1分
const MAX_REQUESTS = 100;

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const ip = request.headers.get('CF-Connecting-IP') || 'unknown';
    const key = `rate_limit:${ip}`;

    const current = await env.RATE_LIMIT.get(key);
    const count = current ? parseInt(current) : 0;

    if (count >= MAX_REQUESTS) {
      return new Response('Too Many Requests', {
        status: 429,
        headers: {
          'Retry-After': RATE_LIMIT_WINDOW.toString(),
        },
      });
    }

    await env.RATE_LIMIT.put(key, (count + 1).toString(), {
      expirationTtl: RATE_LIMIT_WINDOW,
    });

    return new Response('OK');
  },
};
```

## CSRF保護

### CSRFトークンの検証

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    if (['POST', 'PUT', 'DELETE'].includes(request.method)) {
      const csrfToken = request.headers.get('X-CSRF-Token');
      const cookieHeader = request.headers.get('Cookie') || '';
      const sessionCsrf = cookieHeader
        .split(';')
        .find(c => c.trim().startsWith('csrf='))
        ?.split('=')[1];

      if (!csrfToken || csrfToken !== sessionCsrf) {
        return new Response('CSRF token mismatch', { status: 403 });
      }
    }

    return new Response('OK');
  },
};
```

## XSS対策

### 出力のサニタイズ

```typescript
function escapeHtml(unsafe: string): string {
  return unsafe
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;');
}

export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const userInput = url.searchParams.get('name') || 'Guest';

    // ユーザー入力をエスケープ
    const safeName = escapeHtml(userInput);

    const html = `
      <!DOCTYPE html>
      <html>
        <body>
          <h1>Hello, ${safeName}!</h1>
        </body>
      </html>
    `;

    return new Response(html, {
      headers: { 'Content-Type': 'text/html; charset=utf-8' },
    });
  },
};
```

## SQLインジェクション対策

### パラメータ化クエリ

```typescript
export interface Env {
  DB: D1Database;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const userId = url.searchParams.get('id');

    // ❌ 危険: SQL インジェクション
    // const query = `SELECT * FROM users WHERE id = ${userId}`;

    // ✅ 安全: パラメータ化クエリ
    const { results } = await env.DB.prepare(
      'SELECT * FROM users WHERE id = ?'
    )
      .bind(userId)
      .all();

    return new Response(JSON.stringify(results));
  },
};
```

## Secretsの管理

### 環境変数とSecrets

```bash
# Secretsの設定（機密情報）
wrangler secret put API_KEY
wrangler secret put DATABASE_PASSWORD

# 環境変数の設定（非機密情報）
# wrangler.toml
[vars]
API_VERSION = "v1"
ENVIRONMENT = "production"
```

```typescript
export interface Env {
  API_KEY: string; // Secret
  API_VERSION: string; // 環境変数
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Secretsを使用（コードに含まれない）
    const apiKey = env.API_KEY;

    // 環境変数を使用
    const version = env.API_VERSION;

    return new Response(`API ${version}`);
  },
};
```

## HTTPS強制

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    // HTTPをHTTPSにリダイレクト
    if (url.protocol === 'http:') {
      url.protocol = 'https:';
      return Response.redirect(url.toString(), 301);
    }

    return new Response('Secure connection');
  },
};
```

## セキュリティチェックリスト

- [ ] 認証・認可の実装
- [ ] セキュリティヘッダーの設定
- [ ] 入力検証
- [ ] レート制限
- [ ] CSRF保護
- [ ] XSS対策
- [ ] SQLインジェクション対策
- [ ] Secretsの適切な管理
- [ ] HTTPS強制
- [ ] ログの適切な管理（機密情報を含めない）

## まとめ

セキュリティは多層防御が重要です：

1. **認証・認可**: 適切なアクセス制御
2. **入力検証**: すべての入力を検証
3. **出力エスケープ**: XSS対策
4. **セキュリティヘッダー**: ブラウザの保護機能を活用
5. **レート制限**: DDoS対策
6. **Secrets管理**: 機密情報の適切な扱い

---

このカリキュラムで、Cloudflare Workersの基礎から応用まで学習できます。

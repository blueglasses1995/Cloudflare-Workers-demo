# 02. 公式仕様と技術詳細

## 目次
1. [Runtime環境](#runtime環境)
2. [Runtime API](#runtime-api)
3. [パフォーマンス特性](#パフォーマンス特性)
4. [制限事項](#制限事項)
5. [セキュリティモデル](#セキュリティモデル)

## Runtime環境

### V8 JavaScriptエンジン

Cloudflare WorkersはGoogle V8エンジンをベースにしています。

#### サポートされるJavaScript機能

```javascript
// ES2022までの機能をサポート

// ES6+
const arrow = () => {};
const { destructuring } = obj;
const spread = [...array];
async function asyncFunc() {}

// ES2020
const optional = obj?.property;
const nullish = value ?? 'default';
BigInt(9007199254740991);

// ES2021
const str = 'hello';
str.replaceAll('l', 'L');
Promise.any([p1, p2, p3]);

// ES2022
class PrivateFields {
  #private = 'secret';
}
await import('./module.js'); // Top-level await
```

#### 実行環境の特徴

```javascript
// グローバルオブジェクト
// ❌ Node.js: global, process, require
// ❌ Browser: window, document
// ✅ Workers: self, globalThis

console.log(typeof window);     // undefined
console.log(typeof document);   // undefined
console.log(typeof process);    // undefined
console.log(typeof global);     // undefined
console.log(typeof self);       // object
console.log(typeof globalThis); // object
```

### TypeScriptサポート

```typescript
// Workers向けの型定義が提供されている
// @cloudflare/workers-types

import { Request, Response } from '@cloudflare/workers-types';

export default {
  async fetch(
    request: Request,
    env: Env,
    ctx: ExecutionContext
  ): Promise<Response> {
    return new Response('Hello World!');
  },
};

// 環境変数の型定義
interface Env {
  MY_KV: KVNamespace;
  MY_DURABLE_OBJECT: DurableObjectNamespace;
  MY_SECRET: string;
}
```

### WebAssemblyサポート

```javascript
// WebAssemblyモジュールを実行可能

// wasm-pack等でRustからビルド
import wasm from './module.wasm';

export default {
  async fetch(request) {
    const instance = await WebAssembly.instantiate(wasm);
    const result = instance.exports.calculate(42);
    return new Response(`Result: ${result}`);
  },
};
```

## Runtime API

### 1. FetchEvent API

HTTPリクエストを処理する基本API。

#### Service Worker構文（レガシー）

```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  return new Response('Hello World!');
}
```

#### Modules構文（推奨）

```javascript
export default {
  async fetch(request, env, ctx) {
    return new Response('Hello World!');
  },
};
```

**パラメータ**:
- `request`: Requestオブジェクト
- `env`: 環境変数とバインディング
- `ctx`: 実行コンテキスト

#### FetchEventのメソッド

```javascript
export default {
  async fetch(request, env, ctx) {
    // ctx.waitUntil() - レスポンス後も処理を継続
    ctx.waitUntil(
      logAnalytics(request)
    );

    // ctx.passThroughOnException() - エラー時にオリジンへ
    ctx.passThroughOnException();

    return new Response('OK');
  },
};

async function logAnalytics(request) {
  // レスポンスを返した後も実行される
  await fetch('https://analytics.example.com/log', {
    method: 'POST',
    body: JSON.stringify({ url: request.url }),
  });
}
```

### 2. Request API

Web標準のRequestインターフェース。

```javascript
// Request オブジェクトの作成
const request = new Request('https://example.com', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ data: 'value' }),
});

// プロパティとメソッド
console.log(request.url);         // URL
console.log(request.method);      // HTTPメソッド
console.log(request.headers);     // Headers オブジェクト

// ボディの読み取り（一度だけ）
const json = await request.json();
const text = await request.text();
const blob = await request.blob();
const arrayBuffer = await request.arrayBuffer();
const formData = await request.formData();

// Cloudflare独自プロパティ
console.log(request.cf);          // Cloudflare固有情報
```

#### request.cf プロパティ

```javascript
export default {
  async fetch(request) {
    const cf = request.cf;

    return new Response(JSON.stringify({
      // 地理情報
      country: cf.country,           // JP, US, etc
      city: cf.city,                 // Tokyo, New York, etc
      continent: cf.continent,       // AS, NA, etc
      latitude: cf.latitude,         // 35.6895
      longitude: cf.longitude,       // 139.6917
      postalCode: cf.postalCode,     // 100-0001
      region: cf.region,             // Tokyo, California
      timezone: cf.timezone,         // Asia/Tokyo

      // ネットワーク情報
      asn: cf.asn,                   // AS番号
      colo: cf.colo,                 // データセンターコード

      // TLS/HTTP情報
      tlsVersion: cf.tlsVersion,     // TLSv1.3
      httpProtocol: cf.httpProtocol, // HTTP/2, HTTP/3
    }, null, 2));
  },
};
```

### 3. Response API

Web標準のResponseインターフェース。

```javascript
// シンプルなレスポンス
const response = new Response('Hello World!');

// 詳細な設定
const response = new Response('{"message": "Hello"}', {
  status: 200,
  statusText: 'OK',
  headers: {
    'Content-Type': 'application/json',
    'Cache-Control': 'max-age=3600',
  },
});

// リダイレクト
const redirect = Response.redirect('https://example.com', 302);

// ストリーミングレスポンス
const stream = new ReadableStream({
  start(controller) {
    controller.enqueue('chunk 1\n');
    controller.enqueue('chunk 2\n');
    controller.close();
  },
});
const streaming = new Response(stream);
```

### 4. Headers API

```javascript
// Headers オブジェクト
const headers = new Headers();
headers.set('Content-Type', 'application/json');
headers.append('Set-Cookie', 'session=abc');
headers.append('Set-Cookie', 'user=123');
headers.delete('X-Old-Header');

// 読み取り
console.log(headers.get('Content-Type'));  // application/json
console.log(headers.has('Content-Type'));  // true

// イテレーション
for (const [key, value] of headers) {
  console.log(`${key}: ${value}`);
}

// レスポンスのヘッダー操作
const response = new Response('Hello');
response.headers.set('X-Custom', 'value');
```

### 5. Fetch API

```javascript
// 基本的なfetch
const response = await fetch('https://api.example.com/data');
const data = await response.json();

// オプション指定
const response = await fetch('https://api.example.com/data', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer token',
  },
  body: JSON.stringify({ key: 'value' }),
  // Cloudflare独自オプション
  cf: {
    // キャッシュTTL
    cacheTtl: 300,
    // キャッシュキー
    cacheKey: 'custom-key',
    // リゾルバー
    resolveOverride: 'example.com',
  },
});
```

### 6. Cache API

```javascript
// デフォルトキャッシュ
const cache = caches.default;

// キャッシュの読み取り
const cachedResponse = await cache.match(request);
if (cachedResponse) {
  return cachedResponse;
}

// キャッシュへの書き込み
const response = await fetch(request);
await cache.put(request, response.clone());
return response;

// 名前付きキャッシュ
const myCache = await caches.open('my-cache');
await myCache.put(request, response);
```

### 7. Streams API

```javascript
// ReadableStream
const stream = new ReadableStream({
  async start(controller) {
    controller.enqueue('data chunk 1');
    controller.enqueue('data chunk 2');
    controller.close();
  },
});

// TransformStream
const transformStream = new TransformStream({
  transform(chunk, controller) {
    controller.enqueue(chunk.toUpperCase());
  },
});

// パイプライン
const response = new Response(
  stream.pipeThrough(transformStream)
);

// HTMLリライター（Cloudflare独自）
const rewriter = new HTMLRewriter()
  .on('a', {
    element(element) {
      element.setAttribute('target', '_blank');
    },
  });

return rewriter.transform(response);
```

### 8. Web Crypto API

```javascript
// ランダムなUUID生成
const uuid = crypto.randomUUID();

// ランダムな値生成
const array = new Uint8Array(16);
crypto.getRandomValues(array);

// ハッシュ生成
const data = new TextEncoder().encode('hello');
const hashBuffer = await crypto.subtle.digest('SHA-256', data);
const hashArray = Array.from(new Uint8Array(hashBuffer));
const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');

// HMAC
const key = await crypto.subtle.generateKey(
  { name: 'HMAC', hash: 'SHA-256' },
  false,
  ['sign', 'verify']
);
const signature = await crypto.subtle.sign('HMAC', key, data);
```

### 9. URL API

```javascript
const url = new URL('https://example.com/path?query=value');

console.log(url.protocol);   // https:
console.log(url.hostname);   // example.com
console.log(url.pathname);   // /path
console.log(url.search);     // ?query=value

// URLSearchParams
const params = url.searchParams;
console.log(params.get('query'));  // value
params.set('new', 'param');
params.delete('query');
```

### 10. TextEncoder / TextDecoder

```javascript
// テキストのエンコード
const encoder = new TextEncoder();
const uint8array = encoder.encode('Hello 世界');

// テキストのデコード
const decoder = new TextDecoder('utf-8');
const text = decoder.decode(uint8array);
```

### 11. Scheduled Event（Cron Triggers）

```javascript
export default {
  async scheduled(event, env, ctx) {
    // event.scheduledTime: 実行予定時刻（ミリ秒）
    // event.cron: cron式

    ctx.waitUntil(doScheduledTask());
  },
};

// wrangler.toml での設定
// [triggers]
// crons = ["0 0 * * *"]  # 毎日0時
```

## パフォーマンス特性

### 起動時間

```
コールドスタート: <1ms
ウォームスタート: <0.1ms
```

V8 Isolatesにより、ほぼ瞬時に起動します。

### CPU時間制限

| プラン | CPU時間/リクエスト |
|--------|-------------------|
| Free | 10ms |
| Paid | 50ms (標準) / 無制限（オプション） |

```javascript
// CPU時間の計測
const start = Date.now();
// 重い処理
const duration = Date.now() - start;
console.log(`CPU time: ${duration}ms`);
```

### メモリ制限

| プラン | メモリ制限 |
|--------|-----------|
| Free | 128MB |
| Paid | 128MB (標準) / 256MB (オプション) |

### リクエストサイズ制限

```
リクエストボディ: 最大 100MB (Free), 500MB (Paid)
レスポンスボディ: 制限なし（ストリーミング）
ヘッダーサイズ: 最大 32KB
```

### 同時リクエスト数

```
制限なし（自動スケーリング）
```

### サブリクエスト制限

```javascript
// 1リクエストあたり最大50のサブリクエスト（Free）
// 1リクエストあたり最大1000のサブリクエスト（Paid）

const promises = [];
for (let i = 0; i < 10; i++) {
  promises.push(fetch(`https://api.example.com/data/${i}`));
}
const responses = await Promise.all(promises);
```

## 制限事項

### 利用不可のAPI

```javascript
// ❌ Node.js API
require()           // CommonJS
process.env         // 環境変数（envを使用）
fs                  // ファイルシステム
Buffer              // Node.js Buffer（代わりにUint8Array）

// ❌ Browser-specific API
window
document
localStorage
XMLHttpRequest

// ❌ 時間のかかる操作
setTimeout()        // 非同期実行不可
setInterval()       // 非同期実行不可
```

### タイムアウト

```javascript
// すべての処理は同期的に完了する必要がある
export default {
  async fetch(request, env, ctx) {
    // ✅ 正しい: awaitで待機
    const response = await fetch('https://api.example.com');

    // ❌ 誤り: 非同期処理を放置
    fetch('https://api.example.com'); // 完了を待たない

    // ✅ ctx.waitUntilを使用
    ctx.waitUntil(
      fetch('https://analytics.example.com/log')
    );

    return response;
  },
};
```

### グローバル状態

```javascript
// ⚠️ グローバル変数は共有されない
let counter = 0;

export default {
  async fetch(request) {
    counter++; // リクエスト間で共有されない
    return new Response(`Count: ${counter}`);
  },
};

// ✅ 永続化が必要な場合はKVやDurable Objectsを使用
```

## セキュリティモデル

### 1. Isolate分離

```javascript
// 各リクエストは独立したV8 Isolate内で実行
// メモリは完全に分離

// リクエストA
const secret = "password"; // 他のIsolateから見えない

// リクエストB（別のIsolate）
// secretにはアクセス不可
```

### 2. Content Security Policy

```javascript
// CSPヘッダーの設定
export default {
  async fetch(request) {
    return new Response('Hello', {
      headers: {
        'Content-Security-Policy': "default-src 'self'",
      },
    });
  },
};
```

### 3. CORS制御

```javascript
function handleCORS(request) {
  const corsHeaders = {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
    'Access-Control-Allow-Headers': 'Content-Type',
  };

  if (request.method === 'OPTIONS') {
    return new Response(null, { headers: corsHeaders });
  }

  return corsHeaders;
}
```

### 4. Secrets管理

```javascript
// wrangler.toml
// [vars]
// PUBLIC_KEY = "public-value"  # 公開OK

// wranglerコマンドでSecretsを設定
// wrangler secret put API_KEY

export default {
  async fetch(request, env) {
    // env.API_KEY からアクセス（コードに含まれない）
    const apiKey = env.API_KEY;

    return new Response('OK');
  },
};
```

### 5. レート制限

```javascript
// IPベースのレート制限
const RATE_LIMIT = 100; // リクエスト/分

export default {
  async fetch(request, env) {
    const ip = request.headers.get('CF-Connecting-IP');
    const key = `rate_limit:${ip}`;

    const count = await env.KV.get(key);
    if (count && parseInt(count) > RATE_LIMIT) {
      return new Response('Too Many Requests', { status: 429 });
    }

    await env.KV.put(key, (parseInt(count || 0) + 1).toString(), {
      expirationTtl: 60,
    });

    return new Response('OK');
  },
};
```

### 6. DDoS保護

Cloudflareの自動DDoS保護が適用されます：
- L3/L4レベルの保護
- L7アプリケーション層の保護
- ボット検出

### 7. 暗号化

```javascript
// HTTPS強制
export default {
  async fetch(request) {
    const url = new URL(request.url);

    if (url.protocol === 'http:') {
      url.protocol = 'https:';
      return Response.redirect(url.toString(), 301);
    }

    return new Response('Secure connection');
  },
};
```

## ベストプラクティス

### 1. エラーハンドリング

```javascript
export default {
  async fetch(request, env, ctx) {
    try {
      return await handleRequest(request, env);
    } catch (error) {
      return new Response(`Error: ${error.message}`, {
        status: 500,
      });
    }
  },
};
```

### 2. ストリーミング処理

```javascript
// 大きなレスポンスはストリーミング
export default {
  async fetch(request) {
    const response = await fetch('https://large-file.example.com');
    // そのままストリーミング
    return response;
  },
};
```

### 3. キャッシュ活用

```javascript
export default {
  async fetch(request) {
    const cache = caches.default;
    let response = await cache.match(request);

    if (!response) {
      response = await fetch(request);
      // キャッシュに保存
      await cache.put(request, response.clone());
    }

    return response;
  },
};
```

## まとめ

Cloudflare Workersは：
- **Web標準準拠**: 標準APIを最大限活用
- **高パフォーマンス**: 超高速起動と実行
- **セキュア**: 強固な分離とセキュリティ
- **スケーラブル**: 自動スケーリング

制限事項を理解した上で活用することで、強力なエッジアプリケーションを構築できます。

---

次: [03. 機能一覧と詳細](./03-features.md)

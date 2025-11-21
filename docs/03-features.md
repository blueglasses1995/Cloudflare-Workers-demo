# 03. 機能一覧と詳細

## 目次
1. [コア機能](#コア機能)
2. [Workers KV](#workers-kv)
3. [Durable Objects](#durable-objects)
4. [R2 Storage](#r2-storage)
5. [D1 Database](#d1-database)
6. [Queue](#queue)
7. [Analytics Engine](#analytics-engine)
8. [その他のサービス](#その他のサービス)

## コア機能

### 1. HTTP リクエスト処理

最も基本的な機能。HTTPリクエストを受け取り、レスポンスを返します。

```javascript
export default {
  async fetch(request, env, ctx) {
    return new Response('Hello World!');
  },
};
```

**主な用途**:
- API開発
- リバースプロキシ
- エッジロジック実装
- A/Bテスト
- リダイレクト

### 2. HTMLRewriter

HTMLをストリーミングで書き換える機能。

```javascript
export default {
  async fetch(request) {
    const response = await fetch(request);

    return new HTMLRewriter()
      .on('a', {
        element(element) {
          // すべての<a>タグにtarget="_blank"を追加
          element.setAttribute('target', '_blank');
        },
      })
      .on('title', {
        element(element) {
          element.setInnerContent('Modified Title');
        },
      })
      .transform(response);
  },
};
```

**主な用途**:
- 広告挿入
- アナリティクストラッキング追加
- セキュリティヘッダー追加
- レスポンシブ対応
- 多言語化

### 3. Cache API

エッジでのキャッシング機能。

```javascript
export default {
  async fetch(request) {
    const cache = caches.default;

    // キャッシュを確認
    let response = await cache.match(request);

    if (!response) {
      // キャッシュミス: オリジンから取得
      response = await fetch(request);

      // Cache-Controlヘッダーに従ってキャッシュ
      const headers = new Headers(response.headers);
      headers.set('Cache-Control', 'max-age=3600');

      response = new Response(response.body, {
        status: response.status,
        statusText: response.statusText,
        headers: headers,
      });

      // キャッシュに保存
      await cache.put(request, response.clone());
    }

    return response;
  },
};
```

**主な用途**:
- 静的コンテンツのキャッシュ
- API レスポンスのキャッシュ
- パフォーマンス最適化

### 4. Cron Triggers

定期実行機能。

```javascript
export default {
  async scheduled(event, env, ctx) {
    // 毎時0分に実行される処理
    await cleanupOldData(env);
    await generateReports(env);
  },
};
```

**wrangler.toml設定**:
```toml
[triggers]
crons = [
  "0 * * * *",      # 毎時0分
  "0 0 * * *",      # 毎日0時
  "0 0 * * 1",      # 毎週月曜0時
  "*/15 * * * *"    # 15分ごと
]
```

**主な用途**:
- データクリーンアップ
- レポート生成
- データ同期
- ヘルスチェック

## Workers KV

グローバルに分散された低レイテンシのキーバリューストア。

### 特徴

- **グローバルレプリケーション**: 世界中のエッジロケーションに自動複製
- **結果整合性**: 書き込み後、数秒で全エッジに反映
- **低レイテンシ読み込み**: 最寄りのエッジから高速読み込み
- **大容量**: 1キーあたり25MBまで

### 基本操作

```javascript
export default {
  async fetch(request, env) {
    // 読み取り
    const value = await env.MY_KV.get('key');
    const jsonValue = await env.MY_KV.get('key', { type: 'json' });
    const arrayBuffer = await env.MY_KV.get('key', { type: 'arrayBuffer' });
    const stream = await env.MY_KV.get('key', { type: 'stream' });

    // 書き込み
    await env.MY_KV.put('key', 'value');
    await env.MY_KV.put('key', JSON.stringify({ data: 'value' }));

    // 有効期限付き書き込み
    await env.MY_KV.put('key', 'value', {
      expirationTtl: 3600, // 1時間後に削除
    });

    // 削除
    await env.MY_KV.delete('key');

    // リスト取得
    const list = await env.MY_KV.list({ prefix: 'user:' });
    for (const key of list.keys) {
      console.log(key.name);
    }

    return new Response('OK');
  },
};
```

### メタデータ

```javascript
// メタデータ付きで保存
await env.MY_KV.put('user:123', JSON.stringify({ name: 'Alice' }), {
  metadata: { created: Date.now(), type: 'user' },
});

// メタデータ付きで取得
const { value, metadata } = await env.MY_KV.getWithMetadata('user:123');
console.log(metadata.created);
```

### 使用例: セッション管理

```javascript
export default {
  async fetch(request, env) {
    const sessionId = request.headers.get('Cookie')?.match(/session=([^;]+)/)?.[1];

    if (!sessionId) {
      return new Response('Unauthorized', { status: 401 });
    }

    // セッションデータを取得
    const sessionData = await env.SESSIONS.get(`session:${sessionId}`, {
      type: 'json',
    });

    if (!sessionData) {
      return new Response('Session expired', { status: 401 });
    }

    return new Response(`Hello ${sessionData.username}!`);
  },
};
```

### 使用例: キャッシュ

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const cacheKey = `cache:${url.pathname}`;

    // KVからキャッシュを取得
    let cached = await env.CACHE.get(cacheKey);

    if (cached) {
      return new Response(cached, {
        headers: { 'X-Cache': 'HIT' },
      });
    }

    // キャッシュミス: データを取得
    const data = await fetchExpensiveData();

    // KVにキャッシュ（1時間）
    await env.CACHE.put(cacheKey, data, {
      expirationTtl: 3600,
    });

    return new Response(data, {
      headers: { 'X-Cache': 'MISS' },
    });
  },
};
```

### 制限事項

- **書き込みレート**: 1キーあたり1回/秒推奨
- **結果整合性**: 書き込み後、数秒で反映
- **サイズ制限**: 1キー25MB、1ネームスペース無制限
- **リスト操作**: 最大1000キーまで一度に取得

## Durable Objects

強い整合性を持つステートフルなコンピューティング。

### 特徴

- **強い整合性**: 単一インスタンスで状態を保持
- **WebSocketサポート**: リアルタイム通信
- **永続ストレージ**: 状態の永続化
- **自動マイグレーション**: 負荷に応じて最適な場所に移動

### 基本構造

```javascript
// Durable Object クラス
export class Counter {
  constructor(state, env) {
    this.state = state;
    this.count = 0;
  }

  async fetch(request) {
    // 状態の読み込み
    const stored = await this.state.storage.get('count');
    this.count = stored || 0;

    const url = new URL(request.url);

    if (url.pathname === '/increment') {
      this.count++;
      await this.state.storage.put('count', this.count);
      return new Response(`Count: ${this.count}`);
    }

    if (url.pathname === '/get') {
      return new Response(`Count: ${this.count}`);
    }

    return new Response('Not found', { status: 404 });
  }
}

// Worker
export default {
  async fetch(request, env) {
    // Durable Object インスタンスの取得
    const id = env.COUNTER.idFromName('global');
    const stub = env.COUNTER.get(id);

    // Durable Object へリクエスト転送
    return stub.fetch(request);
  },
};
```

### Storage API

```javascript
export class MyDurableObject {
  constructor(state, env) {
    this.state = state;
  }

  async fetch(request) {
    // 単一キーの操作
    await this.state.storage.put('key', 'value');
    const value = await this.state.storage.get('key');
    await this.state.storage.delete('key');

    // 複数キーの操作
    await this.state.storage.put({
      'key1': 'value1',
      'key2': 'value2',
      'key3': 'value3',
    });

    const values = await this.state.storage.get(['key1', 'key2']);

    // リスト取得
    const map = await this.state.storage.list();
    for (const [key, value] of map) {
      console.log(key, value);
    }

    // トランザクション
    await this.state.storage.transaction(async txn => {
      const count = (await txn.get('count')) || 0;
      await txn.put('count', count + 1);
    });

    return new Response('OK');
  }
}
```

### WebSocketの例

```javascript
export class ChatRoom {
  constructor(state, env) {
    this.state = state;
    this.sessions = [];
  }

  async fetch(request) {
    // WebSocket アップグレード
    const upgradeHeader = request.headers.get('Upgrade');
    if (upgradeHeader !== 'websocket') {
      return new Response('Expected WebSocket', { status: 426 });
    }

    const [client, server] = Object.values(new WebSocketPair());

    await this.handleSession(server);

    return new Response(null, {
      status: 101,
      webSocket: client,
    });
  }

  async handleSession(webSocket) {
    webSocket.accept();
    this.sessions.push(webSocket);

    webSocket.addEventListener('message', event => {
      // すべてのセッションにブロードキャスト
      this.broadcast(event.data);
    });

    webSocket.addEventListener('close', () => {
      this.sessions = this.sessions.filter(s => s !== webSocket);
    });
  }

  broadcast(message) {
    for (const session of this.sessions) {
      try {
        session.send(message);
      } catch (err) {
        // 接続が切れている場合は無視
      }
    }
  }
}
```

### 使用例: レート制限

```javascript
export class RateLimiter {
  constructor(state, env) {
    this.state = state;
  }

  async fetch(request) {
    const now = Date.now();
    const requests = (await this.state.storage.get('requests')) || [];

    // 1分以内のリクエストのみ保持
    const recentRequests = requests.filter(time => now - time < 60000);

    // レート制限チェック（60req/min）
    if (recentRequests.length >= 60) {
      return new Response('Too Many Requests', { status: 429 });
    }

    // 新しいリクエストを記録
    recentRequests.push(now);
    await this.state.storage.put('requests', recentRequests);

    return new Response('OK');
  }
}
```

## R2 Storage

S3互換のオブジェクトストレージ。エグレス料金なし。

### 特徴

- **S3互換**: 既存のS3ツールが使用可能
- **エグレス無料**: データ転送料金なし
- **グローバル配信**: Cloudflareネットワーク経由で配信
- **大容量**: 無制限のストレージ

### 基本操作

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // オブジェクトの取得
    if (request.method === 'GET') {
      const object = await env.MY_BUCKET.get(url.pathname);

      if (object === null) {
        return new Response('Not found', { status: 404 });
      }

      return new Response(object.body, {
        headers: {
          'Content-Type': object.httpMetadata.contentType,
          'ETag': object.httpEtag,
        },
      });
    }

    // オブジェクトのアップロード
    if (request.method === 'PUT') {
      await env.MY_BUCKET.put(url.pathname, request.body, {
        httpMetadata: {
          contentType: request.headers.get('Content-Type'),
        },
        customMetadata: {
          uploadedBy: 'worker',
          uploadedAt: new Date().toISOString(),
        },
      });

      return new Response('Uploaded', { status: 201 });
    }

    // オブジェクトの削除
    if (request.method === 'DELETE') {
      await env.MY_BUCKET.delete(url.pathname);
      return new Response('Deleted', { status: 204 });
    }

    return new Response('Method not allowed', { status: 405 });
  },
};
```

### リスト取得

```javascript
export default {
  async fetch(request, env) {
    // バケット内のオブジェクトをリスト
    const listed = await env.MY_BUCKET.list({
      prefix: 'images/',
      limit: 100,
    });

    const objects = listed.objects.map(obj => ({
      key: obj.key,
      size: obj.size,
      uploaded: obj.uploaded,
    }));

    return new Response(JSON.stringify(objects), {
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
```

### 署名付きURL

```javascript
// 一時的なアクセスを許可するURL生成
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const key = url.searchParams.get('key');

    // R2から取得
    const object = await env.MY_BUCKET.get(key);

    if (!object) {
      return new Response('Not found', { status: 404 });
    }

    // 一時URLを生成（実際の実装は異なる）
    const signedUrl = await generateSignedUrl(key, 3600);

    return Response.redirect(signedUrl, 302);
  },
};
```

## D1 Database

エッジで実行できるSQLiteデータベース。

### 特徴

- **SQLite**: 標準SQL
- **エッジ実行**: 低レイテンシクエリ
- **自動レプリケーション**: グローバル配信
- **トランザクションサポート**: ACID特性

### 基本操作

```javascript
export default {
  async fetch(request, env) {
    // SELECT クエリ
    const { results } = await env.DB.prepare(
      'SELECT * FROM users WHERE id = ?'
    ).bind(1).all();

    // INSERT
    await env.DB.prepare(
      'INSERT INTO users (name, email) VALUES (?, ?)'
    ).bind('Alice', 'alice@example.com').run();

    // UPDATE
    await env.DB.prepare(
      'UPDATE users SET name = ? WHERE id = ?'
    ).bind('Bob', 1).run();

    // DELETE
    await env.DB.prepare(
      'DELETE FROM users WHERE id = ?'
    ).bind(1).run();

    return new Response(JSON.stringify(results));
  },
};
```

### バッチ処理

```javascript
export default {
  async fetch(request, env) {
    // 複数のクエリを一度に実行
    const results = await env.DB.batch([
      env.DB.prepare('INSERT INTO users (name) VALUES (?)').bind('Alice'),
      env.DB.prepare('INSERT INTO users (name) VALUES (?)').bind('Bob'),
      env.DB.prepare('SELECT * FROM users'),
    ]);

    return new Response(JSON.stringify(results));
  },
};
```

### トランザクション風の操作

```javascript
export default {
  async fetch(request, env) {
    try {
      await env.DB.batch([
        env.DB.prepare('UPDATE accounts SET balance = balance - ? WHERE id = ?')
          .bind(100, 1),
        env.DB.prepare('UPDATE accounts SET balance = balance + ? WHERE id = ?')
          .bind(100, 2),
      ]);

      return new Response('Transaction successful');
    } catch (error) {
      return new Response('Transaction failed', { status: 500 });
    }
  },
};
```

## Queue

非同期メッセージ処理のためのキューシステム。

### 特徴

- **非同期処理**: 重い処理をバックグラウンドで実行
- **バッチ処理**: 複数メッセージを一度に処理
- **自動リトライ**: 失敗時の自動再試行
- **デッドレターキュー**: 処理失敗メッセージの保存

### メッセージ送信

```javascript
export default {
  async fetch(request, env) {
    // キューにメッセージを送信
    await env.MY_QUEUE.send({
      userId: 123,
      action: 'send_email',
      data: { to: 'user@example.com', subject: 'Hello' },
    });

    // バッチ送信
    await env.MY_QUEUE.sendBatch([
      { body: { task: 'task1' } },
      { body: { task: 'task2' } },
      { body: { task: 'task3' } },
    ]);

    return new Response('Queued');
  },
};
```

### メッセージ処理

```javascript
export default {
  async queue(batch, env) {
    // バッチ内のメッセージを処理
    for (const message of batch.messages) {
      try {
        await processMessage(message.body);
        message.ack(); // 成功
      } catch (error) {
        message.retry(); // リトライ
      }
    }
  },
};

async function processMessage(body) {
  // 実際の処理
  console.log('Processing:', body);
}
```

## Analytics Engine

カスタム分析データの収集と集計。

### 特徴

- **リアルタイム収集**: 低オーバーヘッドでデータ収集
- **SQL分析**: SQLクエリでデータ分析
- **長期保存**: データの長期保存
- **低コスト**: 大量データの収集に最適

### データ書き込み

```javascript
export default {
  async fetch(request, env, ctx) {
    // アナリティクスデータを記録
    ctx.waitUntil(
      env.ANALYTICS.writeDataPoint({
        blobs: [request.url, request.cf.country],
        doubles: [performance.now()],
        indexes: [request.cf.colo],
      })
    );

    return new Response('OK');
  },
};
```

### データ分析（GraphQL API）

```graphql
query {
  viewer {
    accounts(filter: { accountTag: "YOUR_ACCOUNT_ID" }) {
      analyticsEngine {
        dataset(name: "DATASET_NAME") {
          data(
            filter: {
              datetime_geq: "2024-01-01T00:00:00Z"
              datetime_lt: "2024-01-02T00:00:00Z"
            }
          ) {
            blob1
            count
          }
        }
      }
    }
  }
}
```

## その他のサービス

### 1. Workers AI

エッジでAIモデルを実行。

```javascript
export default {
  async fetch(request, env) {
    const response = await env.AI.run('@cf/meta/llama-2-7b-chat-int8', {
      prompt: 'What is the capital of France?',
    });

    return new Response(JSON.stringify(response));
  },
};
```

### 2. Vectorize

ベクトル検索データベース。

```javascript
export default {
  async fetch(request, env) {
    // ベクトルを挿入
    await env.VECTORIZE.insert([
      { id: '1', values: [0.1, 0.2, 0.3], metadata: { text: 'hello' } },
    ]);

    // 類似検索
    const results = await env.VECTORIZE.query([0.1, 0.2, 0.3], {
      topK: 5,
    });

    return new Response(JSON.stringify(results));
  },
};
```

### 3. Workers for Platforms

マルチテナントプラットフォーム構築。

```javascript
// カスタマーのWorkerを動的に実行
export default {
  async fetch(request, env) {
    const customerId = request.headers.get('X-Customer-ID');

    // カスタマー固有のWorkerを実行
    const worker = env.DISPATCH_NAMESPACE.get(customerId);
    return worker.fetch(request);
  },
};
```

## まとめ

Cloudflare Workersは豊富な機能を提供：

| 機能 | 用途 | 特徴 |
|------|------|------|
| Workers KV | キーバリューストア | 低レイテンシ、グローバル配信 |
| Durable Objects | ステートフル処理 | 強い整合性、WebSocket |
| R2 | オブジェクトストレージ | エグレス無料、S3互換 |
| D1 | SQLデータベース | SQLite、エッジ実行 |
| Queue | 非同期処理 | バッチ処理、自動リトライ |
| Analytics Engine | カスタム分析 | リアルタイム収集 |

これらを組み合わせることで、強力なエッジアプリケーションを構築できます。

---

次: ハンズオン編へ

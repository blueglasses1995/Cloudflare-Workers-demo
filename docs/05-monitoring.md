# 05. モニタリングとデバッグ

## ログの活用

### console.log

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    console.log('Request received:', {
      method: request.method,
      url: request.url,
      headers: Object.fromEntries(request.headers),
    });

    return new Response('OK');
  },
};
```

### リアルタイムログ

```bash
wrangler tail
```

出力のフィルタリング：

```bash
# ステータスコードでフィルタ
wrangler tail --status 404

# メソッドでフィルタ
wrangler tail --method POST
```

## Analytics

### Cloudflare Dashboard

Workers & Pages → Analytics で確認できる情報：
- リクエスト数
- エラー率
- CPU時間
- レイテンシ

### Analytics Engine

```typescript
export interface Env {
  ANALYTICS: AnalyticsEngineDataset;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const start = Date.now();

    const response = await handleRequest(request);

    const duration = Date.now() - start;

    // カスタムアナリティクスを記録
    ctx.waitUntil(
      env.ANALYTICS.writeDataPoint({
        blobs: [request.url, response.status.toString()],
        doubles: [duration],
        indexes: [request.cf?.colo],
      })
    );

    return response;
  },
};
```

## エラートラッキング

### Sentry統合

```typescript
import * as Sentry from '@sentry/browser';

export default {
  async fetch(request: Request): Promise<Response> {
    try {
      return await handleRequest(request);
    } catch (error) {
      Sentry.captureException(error);
      return new Response('Internal Server Error', { status: 500 });
    }
  },
};
```

## パフォーマンス測定

### タイミング測定

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const timings: Record<string, number> = {};

    const start = Date.now();

    // データベースクエリ
    const dbStart = Date.now();
    const data = await queryDatabase();
    timings.database = Date.now() - dbStart;

    // 外部API呼び出し
    const apiStart = Date.now();
    const apiResponse = await fetch('https://api.example.com');
    timings.api = Date.now() - apiStart;

    timings.total = Date.now() - start;

    return new Response(JSON.stringify({ data, timings }));
  },
};
```

## デバッグ技術

### ローカルデバッグ

```bash
# デバッグモードで起動
wrangler dev --local

# ポート指定
wrangler dev --port 8788
```

### VS Code デバッグ設定

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Wrangler Dev",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "wrangler",
      "runtimeArgs": ["dev"],
      "skipFiles": ["<node_internals>/**"]
    }
  ]
}
```

## ベストプラクティス

1. **構造化ログ**: JSON形式でログ出力
2. **エラー分類**: エラーレベルを分ける
3. **メトリクス収集**: 重要な指標を記録
4. **アラート設定**: 異常を早期検知
5. **定期的なレビュー**: ログとメトリクスの確認

---

次: [06. パフォーマンス最適化](./06-performance.md)

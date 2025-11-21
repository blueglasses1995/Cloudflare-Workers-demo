# ハンズオン03: ルーティングの実装

## 学習目標

このハンズオンでは、Cloudflare Workersでのルーティングシステムを一から実装します。フレームワークに頼らず、基本から理解することで、より柔軟なアプリケーション開発ができるようになります。

- パスベースのルーティング実装
- 動的ルート（パラメータ）の処理
- クエリパラメータの扱い
- ミドルウェアパターンの実装
- 再利用可能なRouterクラスの作成

## 事前準備

```bash
mkdir routing-demo
cd routing-demo
wrangler init
```

## 📚 カリキュラム構成

このハンズオンは以下のステップで構成されています：

### [Step 1: 基本的なルーティング](./step1-basic-routing.md)
- URLパターンマッチング
- 静的ルートの実装
- HTTPメソッドの処理
- レスポンスの返却

**所要時間**: 20分

### [Step 2: 動的ルート](./step2-dynamic-routes.md)
- パスパラメータの抽出
- 正規表現パターンマッチング
- パラメータのバリデーション
- 複数パラメータの処理

**所要時間**: 25分

### [Step 3: ミドルウェアパターン](./step3-middleware.md)
- ミドルウェアの概念
- リクエスト前処理
- レスポンス後処理
- エラーハンドリングミドルウェア

**所要時間**: 30分

### [Step 4: Routerクラスの実装](./step4-router-class.md)
- 再利用可能なRouterクラス
- ルート登録の抽象化
- ネストされたルート
- 実践的なAPI設計

**所要時間**: 30分

## 完成イメージ

このハンズオンを完了すると、以下のようなルーティングシステムが実装できます：

```typescript
const router = new Router();

// 静的ルート
router.get('/', handleHome);
router.get('/about', handleAbout);

// 動的ルート
router.get('/users/:id', handleUser);
router.get('/posts/:postId/comments/:commentId', handleComment);

// ミドルウェア
router.use(loggingMiddleware);
router.use(authMiddleware);

// ルートグループ
router.group('/api', () => {
  router.get('/users', listUsers);
  router.post('/users', createUser);
});

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    return router.handle(request, env);
  },
};
```

## 学習の進め方

1. **Step 1から順番に進める**: 各ステップは前のステップの知識を基に構築されています
2. **コードを実際に書く**: コピペではなく、理解しながらタイピングしてください
3. **動作確認を忘れずに**: 各ステップで必ず`wrangler dev`で動作確認
4. **演習問題に挑戦**: 理解を深めるために演習問題を解いてください

## 必要な前提知識

- [ハンズオン01: 環境構築とHello World](../01-getting-started/README.md)
- [ハンズオン02: リクエスト/レスポンス処理](../02-request-response/README.md)
- JavaScriptの基本文法
- 正規表現の基礎

## よくある質問

**Q: フレームワークを使わないのはなぜ？**
A: 基礎を理解することで、既存のフレームワーク（Hono, itty-router等）をより深く理解でき、カスタマイズも容易になります。

**Q: 本番環境でもこのコードを使える？**
A: Step 4で作成するRouterクラスは本番環境でも使用可能です。ただし、より高機能なフレームワークの使用も検討してください。

**Q: TypeScriptは必須？**
A: このハンズオンではTypeScriptを使用しますが、JavaScriptでも同様に実装可能です。

## 次のステップ

全てのステップを完了したら：
- [ハンズオン04: Workers KVでのデータ管理](../04-workers-kv/README.md)
- [ハンズオン05: REST API開発](../05-rest-api/README.md)

## 参考リソース

- [Cloudflare Workers Routing Patterns](https://developers.cloudflare.com/workers/examples/)
- [itty-router](https://github.com/kwhitley/itty-router) - 人気のWorkers向けルーター
- [Hono](https://hono.dev/) - 高速なWebフレームワーク

---

それでは、[Step 1: 基本的なルーティング](./step1-basic-routing.md)から始めましょう！

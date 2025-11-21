# ハンズオン06: 外部API連携

## 学習目標

このハンズオンでは、Cloudflare Workersから外部APIを効率的に呼び出す方法を学びます。

- 外部APIの基本的な呼び出し
- エラーハンドリングとリトライ戦略
- レスポンスのキャッシング
- タイムアウト処理
- API集約パターン
- 複数APIの並列呼び出し

## 事前準備

```bash
mkdir external-api-demo
cd external-api-demo
wrangler init
```

## 📚 カリキュラム構成

このハンズオンは以下のステップで構成されています：

### [Step 1: 基本的な外部API呼び出し](./step1-basic-fetch.md)
- Fetch APIの基本
- JSONレスポンスの処理
- クエリパラメータの扱い
- HTTPヘッダーの設定

**所要時間**: 20分

### [Step 2: エラーハンドリングとリトライ](./step2-error-handling.md)
- ネットワークエラーの処理
- HTTPエラーステータスの処理
- リトライロジックの実装
- エクスポネンシャルバックオフ

**所要時間**: 30分

### [Step 3: キャッシング戦略](./step3-caching.md)
- Workers KVでのキャッシング
- Cache APIの活用
- キャッシュの有効期限管理
- キャッシュキーの設計

**所要時間**: 30分

### [Step 4: タイムアウトと並列処理](./step4-timeout-parallel.md)
- AbortControllerによるタイムアウト
- Promise.allでの並列処理
- Promise.raceの活用
- パフォーマンス最適化

**所要時間**: 25分

### [Step 5: 実践プロジェクト](./step5-practical-project.md)
- 天気情報API集約サービス
- 複数APIの統合
- エラーハンドリングの統合
- プロダクションレディなコード

**所要時間**: 40分

## 完成イメージ

このハンズオンを完了すると、以下のような堅牢なAPI呼び出しシステムが実装できます：

```typescript
const apiClient = new APIClient({
  baseURL: 'https://api.example.com',
  timeout: 5000,
  retries: 3,
  cache: true,
  cacheTTL: 3600,
});

// 自動リトライ、タイムアウト、キャッシング付き
const data = await apiClient.get('/users/123');
```

## 使用する公開API

このハンズオンでは以下の無料APIを使用します：

- [JSONPlaceholder](https://jsonplaceholder.typicode.com/) - テスト用REST API
- [Open-Meteo](https://open-meteo.com/) - 天気情報API（無料、APIキー不要）
- [ExchangeRate-API](https://www.exchangerate-api.com/) - 為替レートAPI

## 学習の進め方

1. **Step 1から順番に進める**: 基礎から徐々に高度な内容へ
2. **実際にAPIを呼び出す**: 本物のAPIを使って学ぶ
3. **エラーケースを試す**: わざとエラーを発生させて動作確認
4. **パフォーマンスを測定**: レスポンス時間を計測してキャッシングの効果を確認

## 必要な前提知識

- [ハンズオン01: 環境構築とHello World](../01-getting-started/README.md)
- [ハンズオン02: リクエスト/レスポンス処理](../02-request-response/README.md)
- [ハンズオン04: Workers KVでのデータ管理](../04-workers-kv/README.md)
- Promiseと非同期処理の基礎

## よくある質問

**Q: 外部APIの呼び出しに制限はある？**
A: Freeプランでは1リクエストあたり50のサブリクエストまで、Paidプランでは1000までです。

**Q: タイムアウトの推奨値は？**
A: 通常は3-5秒程度が推奨です。CPU時間制限も考慮してください。

**Q: どのAPIが使える？**
A: CORSに対応していれば、ほとんどの公開APIが使用可能です。

## 次のステップ

全てのステップを完了したら：
- [ハンズオン07: 認証の実装](../07-authentication/README.md)
- [ハンズオン08: Durable Objectsでリアルタイム通信](../08-durable-objects/README.md)

## 参考リソース

- [Fetch API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [Cloudflare Workers - Make fetch requests](https://developers.cloudflare.com/workers/runtime-apis/fetch/)
- [Public APIs List](https://github.com/public-apis/public-apis)

---

それでは、[Step 1: 基本的な外部API呼び出し](./step1-basic-fetch.md)から始めましょう！

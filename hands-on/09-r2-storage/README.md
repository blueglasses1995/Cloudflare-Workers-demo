# ハンズオン09: R2でファイルストレージ

## 学習目標
- R2 Storageの基本操作
- ファイルのアップロード/ダウンロード
- メタデータの管理
- ストリーミング処理

## カリキュラム構成

### [Step 1: R2の基本操作](./step1-basics.md)
- R2バケットの作成
- ファイルのアップロード
- ファイルの取得と削除

### [Step 2: メタデータとリスト操作](./step2-metadata.md)
- カスタムメタデータ
- ファイルリストの取得
- フィルタリングと検索

### [Step 3: 実践プロジェクト](./step3-practical-project.md)
- 画像アップローダー
- ファイル共有サービス
- CDN統合

## wrangler.toml設定

\`\`\`toml
[[r2_buckets]]
binding = "MY_BUCKET"
bucket_name = "my-bucket"
\`\`\`

👉 [Step 1: R2の基本操作](./step1-basics.md)

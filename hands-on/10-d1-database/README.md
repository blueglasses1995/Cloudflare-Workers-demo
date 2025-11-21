# ハンズオン10: D1データベース活用

## 学習目標
- D1 Databaseの基本操作
- SQLクエリの実行
- マイグレーション管理
- トランザクション処理

## カリキュラム構成

### [Step 1: D1の基礎とセットアップ](./step1-setup.md)
- D1データベースの作成
- スキーマ定義
- 初期データ投入

### [Step 2: CRUD操作](./step2-crud.md)
- SELECT, INSERT, UPDATE, DELETE
- パラメータ化クエリ
- バッチ処理

### [Step 3: 実践プロジェクト](./step3-practical-project.md)
- ブログシステム
- ユーザー管理
- リレーションと JOIN

## wrangler.toml設定

\`\`\`toml
[[d1_databases]]
binding = "DB"
database_name = "my-database"
database_id = "your-database-id"
\`\`\`

## データベース作成

\`\`\`bash
wrangler d1 create my-database
wrangler d1 execute my-database --file=schema.sql
\`\`\`

👉 [Step 1: D1の基礎とセットアップ](./step1-setup.md)

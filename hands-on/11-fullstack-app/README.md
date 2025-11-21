# ハンズオン11: フルスタックアプリケーション

## 学習目標
- フロントエンドとバックエンドの統合
- Pages Functionsの活用
- 認証フロー全体の実装
- 本番デプロイメント

## カリキュラム構成

### [Step 1: プロジェクト構成](./step1-project-structure.md)
- ディレクトリ構造
- フロントエンド（React/Vue/Svelte）のセットアップ
- Workersの統合

### [Step 2: 認証システム](./step2-authentication.md)
- ログイン/ログアウト
- セッション管理
- 保護されたルート

### [Step 3: API実装](./step3-api-implementation.md)
- RESTful API
- データベース統合
- ファイルアップロード

### [Step 4: デプロイメント](./step4-deployment.md)
- CI/CDパイプライン
- 環境変数管理
- モニタリング

## 技術スタック

- Frontend: React + TypeScript
- Backend: Cloudflare Workers
- Database: D1
- Storage: R2
- Auth: JWT + Sessions
- Deployment: Cloudflare Pages

## プロジェクト例

このハンズオンでは、以下のようなフルスタックアプリケーションを構築します：

- **タスク管理アプリ**
  - ユーザー認証
  - タスクのCRUD
  - ファイル添付
  - リアルタイム更新

## 完成イメージ

\`\`\`typescript
// Worker (Backend)
export default {
  async fetch(request, env, ctx) {
    return router.handle(request, env, ctx);
  },
};

// Frontend (React)
function App() {
  const { user, login, logout } = useAuth();
  const { tasks, createTask } = useTasks();

  return (
    <div>
      {user ? <Dashboard tasks={tasks} /> : <Login onLogin={login} />}
    </div>
  );
}
\`\`\`

👉 [Step 1: プロジェクト構成](./step1-project-structure.md)

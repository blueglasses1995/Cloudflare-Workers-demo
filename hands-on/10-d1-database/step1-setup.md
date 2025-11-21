# Step 1: D1の基礎とセットアップ

## スキーマ定義

\`\`\`sql
-- schema.sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE posts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id INTEGER NOT NULL,
  title TEXT NOT NULL,
  content TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);
\`\`\`

## 基本的なクエリ

\`\`\`typescript
interface Env {
  DB: D1Database;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // SELECT
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
    ).bind('Alice Updated', 1).run();

    // DELETE
    await env.DB.prepare(
      'DELETE FROM users WHERE id = ?'
    ).bind(1).run();

    return new Response(JSON.stringify(results), {
      headers: { 'Content-Type': 'application/json' },
    });
  },
};
\`\`\`

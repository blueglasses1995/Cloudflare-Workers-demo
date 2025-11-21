# ハンズオン05: REST API開発

## 学習目標

- フルスタックなREST APIの構築
- ルーティングフレームワークの実装
- バリデーション
- エラーハンドリング
- 本番レベルのAPI設計

## 事前準備

```bash
mkdir todo-api
cd todo-api
wrangler init
```

### KVネームスペースの作成

```bash
wrangler kv:namespace create "TODOS"
wrangler kv:namespace create "TODOS" --preview
```

### wrangler.tomlに追加

```toml
name = "todo-api"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[[kv_namespaces]]
binding = "TODOS"
id = "your-namespace-id"
preview_id = "your-preview-id"
```

## Step 1: プロジェクト構造

推奨ディレクトリ構造：

```
src/
├── index.ts          # エントリーポイント
├── router.ts         # ルーター
├── handlers/         # リクエストハンドラー
│   └── todos.ts
├── models/          # データモデル
│   └── todo.ts
├── utils/           # ユーティリティ
│   ├── response.ts
│   └── validation.ts
└── types/           # 型定義
    └── index.ts
```

## Step 2: 型定義

### src/types/index.ts

```typescript
export interface Todo {
  id: string;
  title: string;
  description: string;
  completed: boolean;
  createdAt: string;
  updatedAt: string;
}

export interface CreateTodoRequest {
  title: string;
  description?: string;
}

export interface UpdateTodoRequest {
  title?: string;
  description?: string;
  completed?: boolean;
}

export interface Env {
  TODOS: KVNamespace;
}
```

## Step 3: ユーティリティ関数

### src/utils/response.ts

```typescript
export class ApiResponse {
  static success(data: any, status: number = 200): Response {
    return new Response(JSON.stringify(data), {
      status,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
      },
    });
  }

  static error(message: string, status: number = 400): Response {
    return new Response(
      JSON.stringify({
        error: message,
        status,
      }),
      {
        status,
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*',
        },
      }
    );
  }

  static notFound(resource: string = 'Resource'): Response {
    return this.error(`${resource} not found`, 404);
  }

  static methodNotAllowed(): Response {
    return this.error('Method not allowed', 405);
  }

  static badRequest(message: string): Response {
    return this.error(message, 400);
  }

  static internalError(message: string = 'Internal server error'): Response {
    return this.error(message, 500);
  }
}
```

### src/utils/validation.ts

```typescript
import { CreateTodoRequest, UpdateTodoRequest } from '../types';

export class Validator {
  static validateCreateTodo(data: any): {
    valid: boolean;
    errors: string[];
  } {
    const errors: string[] = [];

    if (!data.title || typeof data.title !== 'string') {
      errors.push('Title is required and must be a string');
    }

    if (data.title && data.title.trim().length === 0) {
      errors.push('Title cannot be empty');
    }

    if (data.title && data.title.length > 200) {
      errors.push('Title must be less than 200 characters');
    }

    if (data.description && typeof data.description !== 'string') {
      errors.push('Description must be a string');
    }

    if (data.description && data.description.length > 1000) {
      errors.push('Description must be less than 1000 characters');
    }

    return {
      valid: errors.length === 0,
      errors,
    };
  }

  static validateUpdateTodo(data: any): {
    valid: boolean;
    errors: string[];
  } {
    const errors: string[] = [];

    if (data.title !== undefined) {
      if (typeof data.title !== 'string') {
        errors.push('Title must be a string');
      } else if (data.title.trim().length === 0) {
        errors.push('Title cannot be empty');
      } else if (data.title.length > 200) {
        errors.push('Title must be less than 200 characters');
      }
    }

    if (data.description !== undefined && typeof data.description !== 'string') {
      errors.push('Description must be a string');
    }

    if (data.completed !== undefined && typeof data.completed !== 'boolean') {
      errors.push('Completed must be a boolean');
    }

    return {
      valid: errors.length === 0,
      errors,
    };
  }
}
```

## Step 4: Todoモデル

### src/models/todo.ts

```typescript
import { Todo, CreateTodoRequest, UpdateTodoRequest, Env } from '../types';

export class TodoModel {
  private env: Env;

  constructor(env: Env) {
    this.env = env;
  }

  async create(data: CreateTodoRequest): Promise<Todo> {
    const id = crypto.randomUUID();
    const now = new Date().toISOString();

    const todo: Todo = {
      id,
      title: data.title,
      description: data.description || '',
      completed: false,
      createdAt: now,
      updatedAt: now,
    };

    await this.env.TODOS.put(`todo:${id}`, JSON.stringify(todo));

    // インデックスに追加
    await this.addToIndex(id);

    return todo;
  }

  async findById(id: string): Promise<Todo | null> {
    const data = await this.env.TODOS.get(`todo:${id}`, { type: 'json' });
    return data as Todo | null;
  }

  async findAll(limit: number = 100): Promise<Todo[]> {
    const indexData = await this.env.TODOS.get('index:todos', { type: 'json' });
    const index = (indexData as string[]) || [];

    const todos = await Promise.all(
      index.slice(0, limit).map(id => this.findById(id))
    );

    return todos.filter((todo): todo is Todo => todo !== null);
  }

  async update(id: string, data: UpdateTodoRequest): Promise<Todo | null> {
    const existing = await this.findById(id);

    if (!existing) {
      return null;
    }

    const updated: Todo = {
      ...existing,
      ...data,
      updatedAt: new Date().toISOString(),
    };

    await this.env.TODOS.put(`todo:${id}`, JSON.stringify(updated));

    return updated;
  }

  async delete(id: string): Promise<boolean> {
    const existing = await this.findById(id);

    if (!existing) {
      return false;
    }

    await this.env.TODOS.delete(`todo:${id}`);
    await this.removeFromIndex(id);

    return true;
  }

  private async addToIndex(id: string): Promise<void> {
    const indexData = await this.env.TODOS.get('index:todos', { type: 'json' });
    const index = (indexData as string[]) || [];

    index.unshift(id); // 最新を先頭に

    await this.env.TODOS.put('index:todos', JSON.stringify(index));
  }

  private async removeFromIndex(id: string): Promise<void> {
    const indexData = await this.env.TODOS.get('index:todos', { type: 'json' });
    const index = (indexData as string[]) || [];

    const filtered = index.filter(i => i !== id);

    await this.env.TODOS.put('index:todos', JSON.stringify(filtered));
  }
}
```

## Step 5: ハンドラー実装

### src/handlers/todos.ts

```typescript
import { Env, CreateTodoRequest, UpdateTodoRequest } from '../types';
import { TodoModel } from '../models/todo';
import { ApiResponse } from '../utils/response';
import { Validator } from '../utils/validation';

export class TodoHandlers {
  private model: TodoModel;

  constructor(env: Env) {
    this.model = new TodoModel(env);
  }

  async list(): Promise<Response> {
    try {
      const todos = await this.model.findAll();
      return ApiResponse.success({ todos, count: todos.length });
    } catch (error) {
      console.error('Error listing todos:', error);
      return ApiResponse.internalError();
    }
  }

  async get(id: string): Promise<Response> {
    try {
      const todo = await this.model.findById(id);

      if (!todo) {
        return ApiResponse.notFound('Todo');
      }

      return ApiResponse.success({ todo });
    } catch (error) {
      console.error('Error getting todo:', error);
      return ApiResponse.internalError();
    }
  }

  async create(request: Request): Promise<Response> {
    try {
      const data = await request.json();

      // バリデーション
      const validation = Validator.validateCreateTodo(data);
      if (!validation.valid) {
        return ApiResponse.badRequest(validation.errors.join(', '));
      }

      const todo = await this.model.create(data as CreateTodoRequest);

      return ApiResponse.success({ todo }, 201);
    } catch (error) {
      console.error('Error creating todo:', error);
      return ApiResponse.internalError();
    }
  }

  async update(id: string, request: Request): Promise<Response> {
    try {
      const data = await request.json();

      // バリデーション
      const validation = Validator.validateUpdateTodo(data);
      if (!validation.valid) {
        return ApiResponse.badRequest(validation.errors.join(', '));
      }

      const todo = await this.model.update(id, data as UpdateTodoRequest);

      if (!todo) {
        return ApiResponse.notFound('Todo');
      }

      return ApiResponse.success({ todo });
    } catch (error) {
      console.error('Error updating todo:', error);
      return ApiResponse.internalError();
    }
  }

  async delete(id: string): Promise<Response> {
    try {
      const deleted = await this.model.delete(id);

      if (!deleted) {
        return ApiResponse.notFound('Todo');
      }

      return ApiResponse.success({ message: 'Todo deleted successfully' });
    } catch (error) {
      console.error('Error deleting todo:', error);
      return ApiResponse.internalError();
    }
  }
}
```

## Step 6: ルーター実装

### src/router.ts

```typescript
import { Env } from './types';
import { TodoHandlers } from './handlers/todos';
import { ApiResponse } from './utils/response';

export class Router {
  private env: Env;
  private todoHandlers: TodoHandlers;

  constructor(env: Env) {
    this.env = env;
    this.todoHandlers = new TodoHandlers(env);
  }

  async handle(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const path = url.pathname;
    const method = request.method;

    // CORS preflight
    if (method === 'OPTIONS') {
      return this.handleCORS();
    }

    // ルーティング
    // GET /todos
    if (path === '/todos' && method === 'GET') {
      return this.todoHandlers.list();
    }

    // POST /todos
    if (path === '/todos' && method === 'POST') {
      return this.todoHandlers.create(request);
    }

    // GET /todos/:id
    const getTodoMatch = path.match(/^\/todos\/([a-zA-Z0-9-]+)$/);
    if (getTodoMatch && method === 'GET') {
      const id = getTodoMatch[1];
      return this.todoHandlers.get(id);
    }

    // PUT /todos/:id
    const putTodoMatch = path.match(/^\/todos\/([a-zA-Z0-9-]+)$/);
    if (putTodoMatch && method === 'PUT') {
      const id = putTodoMatch[1];
      return this.todoHandlers.update(id, request);
    }

    // DELETE /todos/:id
    const deleteTodoMatch = path.match(/^\/todos\/([a-zA-Z0-9-]+)$/);
    if (deleteTodoMatch && method === 'DELETE') {
      const id = deleteTodoMatch[1];
      return this.todoHandlers.delete(id);
    }

    // ルートエンドポイント
    if (path === '/' || path === '') {
      return ApiResponse.success({
        message: 'Todo API',
        version: '1.0.0',
        endpoints: {
          'GET /todos': 'List all todos',
          'POST /todos': 'Create a new todo',
          'GET /todos/:id': 'Get a specific todo',
          'PUT /todos/:id': 'Update a todo',
          'DELETE /todos/:id': 'Delete a todo',
        },
      });
    }

    return ApiResponse.notFound('Endpoint');
  }

  private handleCORS(): Response {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type',
        'Access-Control-Max-Age': '86400',
      },
    });
  }
}
```

## Step 7: メインエントリーポイント

### src/index.ts

```typescript
import { Env } from './types';
import { Router } from './router';
import { ApiResponse } from './utils/response';

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    try {
      const router = new Router(env);
      return await router.handle(request);
    } catch (error) {
      console.error('Unhandled error:', error);
      return ApiResponse.internalError(
        error instanceof Error ? error.message : 'Unknown error'
      );
    }
  },
};
```

## Step 8: テスト

### ローカルで起動

```bash
wrangler dev
```

### cURLでテスト

```bash
# API情報取得
curl http://localhost:8787/

# Todo作成
curl -X POST http://localhost:8787/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn Cloudflare Workers", "description": "Complete all tutorials"}'

# Todo一覧取得
curl http://localhost:8787/todos

# 特定のTodo取得（IDは上記で取得したものを使用）
curl http://localhost:8787/todos/YOUR-TODO-ID

# Todo更新
curl -X PUT http://localhost:8787/todos/YOUR-TODO-ID \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'

# Todo削除
curl -X DELETE http://localhost:8787/todos/YOUR-TODO-ID
```

## Step 9: デプロイ

```bash
wrangler deploy
```

デプロイ後、本番URLでテスト：

```bash
curl https://todo-api.your-subdomain.workers.dev/todos
```

## Step 10: フロントエンド統合例

### HTML + JavaScript

```html
<!DOCTYPE html>
<html>
<head>
  <title>Todo App</title>
  <style>
    body { font-family: Arial, sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
    .todo { padding: 10px; margin: 10px 0; border: 1px solid #ddd; border-radius: 5px; }
    .todo.completed { background: #f0f0f0; text-decoration: line-through; }
    button { margin-left: 10px; }
  </style>
</head>
<body>
  <h1>Todo App</h1>

  <div>
    <input type="text" id="title" placeholder="Todo title" />
    <button onclick="createTodo()">Add Todo</button>
  </div>

  <div id="todos"></div>

  <script>
    const API_URL = 'http://localhost:8787';

    async function fetchTodos() {
      const response = await fetch(`${API_URL}/todos`);
      const data = await response.json();
      displayTodos(data.todos);
    }

    function displayTodos(todos) {
      const container = document.getElementById('todos');
      container.innerHTML = todos.map(todo => `
        <div class="todo ${todo.completed ? 'completed' : ''}">
          <strong>${todo.title}</strong>
          <p>${todo.description}</p>
          <button onclick="toggleTodo('${todo.id}', ${!todo.completed})">
            ${todo.completed ? 'Undo' : 'Complete'}
          </button>
          <button onclick="deleteTodo('${todo.id}')">Delete</button>
        </div>
      `).join('');
    }

    async function createTodo() {
      const title = document.getElementById('title').value;
      if (!title) return;

      await fetch(`${API_URL}/todos`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title })
      });

      document.getElementById('title').value = '';
      fetchTodos();
    }

    async function toggleTodo(id, completed) {
      await fetch(`${API_URL}/todos/${id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ completed })
      });
      fetchTodos();
    }

    async function deleteTodo(id) {
      await fetch(`${API_URL}/todos/${id}`, { method: 'DELETE' });
      fetchTodos();
    }

    // 初回読み込み
    fetchTodos();
  </script>
</body>
</html>
```

## 演習問題

### 問題1: ページネーション

Todo一覧APIにページネーション機能を追加してください。

- クエリパラメータ: `page`, `limit`
- レスポンスに総数と現在のページ情報を含める

### 問題2: 検索機能

タイトルや説明文で検索できる機能を追加してください。

### 問題3: ソート機能

作成日時や更新日時でソートできる機能を追加してください。

## まとめ

このハンズオンで学んだこと：

✅ REST APIの設計と実装
✅ ルーティングシステムの構築
✅ データモデルの分離
✅ バリデーション
✅ エラーハンドリング
✅ CORS対応
✅ 本番レベルのコード構造

## 次のステップ

- [ハンズオン06: 外部API連携](../06-external-api/README.md)
- [ハンズオン07: 認証の実装](../07-authentication/README.md)

## 参考リソース

- [REST API Best Practices](https://restfulapi.net/)
- [Cloudflare Workers Examples](https://developers.cloudflare.com/workers/examples/)

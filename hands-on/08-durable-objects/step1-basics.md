# Step 1: Durable Objectsの基礎

## Durable Objectクラスの定義

\`\`\`typescript
export class Counter {
  private state: DurableObjectState;
  private count: number = 0;

  constructor(state: DurableObjectState) {
    this.state = state;
  }

  async fetch(request: Request): Promise<Response> {
    // ストレージから値を読み込み
    const stored = await this.state.storage.get<number>('count');
    this.count = stored || 0;

    const url = new URL(request.url);

    if (url.pathname === '/increment') {
      this.count++;
      await this.state.storage.put('count', this.count);
      return new Response(\`Count: \${this.count}\`);
    }

    if (url.pathname === '/get') {
      return new Response(\`Count: \${this.count}\`);
    }

    return new Response('Not found', { status: 404 });
  }
}

export default {
  async fetch(request: Request, env: any): Promise<Response> {
    const id = env.COUNTER.idFromName('global');
    const stub = env.COUNTER.get(id);
    return stub.fetch(request);
  },
};
\`\`\`

👉 [Step 2: WebSocket通信](./step2-websocket.md)

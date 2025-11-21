# Step 2: WebSocket通信

## WebSocket対応Durable Object

\`\`\`typescript
export class ChatRoom {
  private state: DurableObjectState;
  private sessions: WebSocket[] = [];

  constructor(state: DurableObjectState) {
    this.state = state;
  }

  async fetch(request: Request): Promise<Response> {
    const upgradeHeader = request.headers.get('Upgrade');
    if (upgradeHeader !== 'websocket') {
      return new Response('Expected WebSocket', { status: 426 });
    }

    const [client, server] = Object.values(new WebSocketPair());
    await this.handleSession(server);

    return new Response(null, { status: 101, webSocket: client });
  }

  async handleSession(webSocket: WebSocket): Promise<void> {
    webSocket.accept();
    this.sessions.push(webSocket);

    webSocket.addEventListener('message', (event) => {
      this.broadcast(event.data);
    });

    webSocket.addEventListener('close', () => {
      this.sessions = this.sessions.filter((s) => s !== webSocket);
    });
  }

  broadcast(message: string): void {
    for (const session of this.sessions) {
      try {
        session.send(message);
      } catch (err) {
        // 接続が切れている場合は無視
      }
    }
  }
}
\`\`\`

👉 [Step 3: リアルタイムチャット](./step3-realtime-chat.md)

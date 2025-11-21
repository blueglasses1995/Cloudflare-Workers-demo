# Step 1: R2の基本操作

## ファイルのアップロード

\`\`\`typescript
interface Env {
  MY_BUCKET: R2Bucket;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method === 'PUT') {
      const url = new URL(request.url);
      const key = url.pathname.slice(1);

      await env.MY_BUCKET.put(key, request.body, {
        httpMetadata: {
          contentType: request.headers.get('Content-Type') || 'application/octet-stream',
        },
      });

      return new Response('Uploaded', { status: 201 });
    }

    if (request.method === 'GET') {
      const url = new URL(request.url);
      const key = url.pathname.slice(1);

      const object = await env.MY_BUCKET.get(key);
      if (!object) {
        return new Response('Not Found', { status: 404 });
      }

      return new Response(object.body, {
        headers: {
          'Content-Type': object.httpMetadata?.contentType || 'application/octet-stream',
        },
      });
    }

    return new Response('Method not allowed', { status: 405 });
  },
};
\`\`\`

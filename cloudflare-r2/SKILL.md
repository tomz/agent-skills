---
name: cloudflare-r2
description: Cloudflare R2 object storage — S3-compatible API, zero egress fees. Buckets, presigned URLs, multipart uploads, lifecycle, Worker bindings.
license: MIT
allowed-tools: read_file, write_file, edit_file, shell, grep, glob
metadata:
  triggers: cloudflare, cloudflare r2, r2, s3, api, bucket, worker
  version: 1.0.0
  updated: 2026-06-26
---
# Cloudflare R2

S3-compatible object storage. **No egress fees** — biggest selling point vs AWS S3.

## CLI setup

```bash
wrangler r2 bucket create my-bucket
wrangler r2 bucket list
wrangler r2 object put my-bucket/key.jpg --file=./local.jpg
wrangler r2 object get my-bucket/key.jpg --file=./out.jpg
wrangler r2 bucket delete my-bucket
```

## From a Worker (binding)

```toml
[[r2_buckets]]
binding = "BUCKET"
bucket_name = "my-bucket"
```

```ts
export default {
  async fetch(req: Request, env: Env): Promise<Response> {
    const url = new URL(req.url);
    const key = url.pathname.slice(1);

    if (req.method === 'PUT') {
      await env.BUCKET.put(key, req.body, {
        httpMetadata: { contentType: req.headers.get('content-type') ?? 'application/octet-stream' },
        customMetadata: { uploadedBy: 'user-123' },
      });
      return new Response(null, { status: 201 });
    }

    const obj = await env.BUCKET.get(key);
    if (!obj) return new Response('Not Found', { status: 404 });

    const headers = new Headers();
    obj.writeHttpMetadata(headers);
    headers.set('etag', obj.httpEtag);
    return new Response(obj.body, { headers });
  },
};
```

## S3-compatible API (any AWS SDK)

```ts
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({
  region: 'auto',
  endpoint: `https://${ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: { accessKeyId: KEY, secretAccessKey: SECRET },
});

const url = await getSignedUrl(
  s3,
  new PutObjectCommand({ Bucket: 'my-bucket', Key: 'avatars/u123.jpg' }),
  { expiresIn: 3600 },
);
```

Get keys: Dashboard → R2 → Manage R2 API Tokens.

## Public buckets & custom domains

Two options:

1. **r2.dev subdomain** (rate-limited; dev only):
   ```bash
   wrangler r2 bucket dev-url enable my-bucket
   ```
2. **Custom domain** (production):
   - Connect a CF-managed domain in bucket settings.
   - Files served through CF cache → effectively a CDN-backed bucket.

## Multipart uploads (>5 GiB single-part limit; up to 5 TiB per object)

```ts
const upload = await env.BUCKET.createMultipartUpload('huge.zip');
const part1 = await upload.uploadPart(1, chunk1);
const part2 = await upload.uploadPart(2, chunk2);
await upload.complete([part1, part2]);
// or upload.abort();
```

## Lifecycle rules

Dashboard → bucket → Settings → Object lifecycle rules. Patterns: expire after N days, abort incomplete multipart after N days.

## Gotchas

- **No SSE-KMS** — encryption is at rest, server-managed only.
- `region: 'auto'` is required for SDK; R2 has no real regions (jurisdiction selectable: default / EU / FedRAMP).
- ETags differ from S3 for multipart objects (R2 uses MD5 of concatenated parts, S3 uses MD5-of-MD5s).
- Free tier: 10 GB-month storage, 1M Class A / 10M Class B ops per month. Object key ≤ 1024 bytes; max object 5 TiB; max single-part upload 5 GiB.
- Up to 1,000,000 buckets per account; 50 custom domains per bucket.

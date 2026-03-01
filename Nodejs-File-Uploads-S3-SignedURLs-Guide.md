# Node.js File Uploads and S3 Signed URLs — Public vs Private Media

A guide to handling file uploads in Node.js, storing them in S3, generating object keys, and serving media securely to the frontend for both public and private content.

---

## 1. Frontend Upload Flow

**Frontend** sends a file to your backend API via `multipart/form-data`.

```http
POST /api/upload
Authorization: Bearer <JWT>
Content-Type: multipart/form-data
```

**Backend** receives the file using Multer or similar, generates a unique object key:

```javascript
const uniqueSuffix = crypto.randomBytes(16).toString('hex');
const key = `private/users/${userId}/signatures/${uniqueSuffix}.png`;
```

Uploads to S3:

```javascript
await s3.putObject({
  Bucket: process.env.S3_BUCKET,
  Key: key,
  Body: fileBuffer,
  ContentType: file.mimetype,
});
```

Stores the `objectKey` (not URL) in the database:

```javascript
await db.files.create({ data: { userId, objectKey: key } });
```

> **Note:** `objectKey` is your canonical reference. Use it to generate access URLs later — never store signed URLs.

---

## 2. Public Media Flow

**Use case:** logos, marketing images, assets accessible by everyone.

* Public folder or public-read S3 bucket
* CDN (e.g., CloudFront) can cache for performance
* No signed URL needed

Store full URL in DB:

```javascript
logoUrl: "https://cdn.example.com/public/orgs/17/logo.png"
```

Or store `objectKey` and build URL on the fly:

```javascript
const publicUrl = `${CDN_URL}/${objectKey}`;
```

Frontend usage:

```jsx
<img src={logoUrl} />
```

Public media flows directly from S3/CDN — no auth, no signing.

---

## 3. Private Media Flow

**Use case:** confidential documents, signatures, contracts, user uploads.

* Private folder or bucket
* Block all public access

Backend:

1. Authenticates the user
2. Fetches `objectKey` from the DB
3. Generates signed URLs per object on demand

```typescript
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { GetObjectCommand } from "@aws-sdk/client-s3";

const signedUrl = await getSignedUrl(
  s3Client,
  new GetObjectCommand({ Bucket: BUCKET, Key: objectKey }),
  { expiresIn: 300 } // 5 minutes
);
```

Returns JSON with signed URLs:

```json
{
  "signatures": [
    { "id": 1, "imageUrl": "https://signed-url..." },
    { "id": 2, "imageUrl": "https://signed-url..." }
  ]
}
```

Frontend usage:

```jsx
<img src={signature.imageUrl} />
```

Signed URLs let the browser fetch directly from S3. Backend does not stream files, so performance scales.

---

## 4. How Signed URLs Work

A signed URL contains:

* Object path (Key)
* Expiration timestamp
* Cryptographic signature (HMAC-SHA256 with AWS secret key)

S3 recomputes the signature internally and checks expiration. If valid, the file is served; otherwise the request is rejected (`403 Forbidden`).

Key points:

* No backend involvement in file serving
* Generating N signed URLs is fast — CPU-only, no network latency
* Revocation via TTL expiration or key rotation

---

## 5. Database Storage Pattern

| Content Type | What to Store |
|---|---|
| Public media | `objectKey` or full URL |
| Private media | `objectKey` only — never the signed URL |

```json
{
  "userId": 42,
  "objectKey": "private/users/42/signatures/abc123.png"
}
```

Generate signed URLs on-demand. Never persist them.

---

## 6. Frontend Fetch Flow Summary

**Public media:**

```
FE → request entity → BE returns public URL → FE fetches directly from CDN/S3
```

**Private media:**

```
FE → request entity → BE authenticates → BE generates signed URLs → FE fetches directly from S3
```

Backend never streams files in the private media scenario — moves the data plane to storage/CDN for scalability.

---

## 7. Key Takeaways

| Rule | Reason |
|---|---|
| Always store `objectKey`, never signed URLs | Signed URLs expire and change |
| Public media: permanent URL, no signing | No auth needed |
| Private media: signed URL per request, short TTL | Temporary, scoped access |
| Use `crypto.randomBytes` for unique filenames | Prevents key collisions |
| Backend + S3/CDN separation | Scales performance |

---

This document serves as a reference for handling file uploads, S3 storage, and secure media access patterns in Node.js APIs.

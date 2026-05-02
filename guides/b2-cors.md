# B2 Bucket CORS Setup

Must be done manually when setting up a new environment (dev or prod). CORS rules are not stored in code or Terraform — they live on the bucket.

## When to run

- Setting up a new B2 bucket
- Bucket CORS was reset or is missing
- Adding new allowed origins

## Buckets

| Environment | Bucket | Key variable | Secret variable |
|-------------|--------|--------------|-----------------|
| Dev | `zeyaroo-dev` | `ZEYAROO_S3_ACCESS_KEY_ID` (Komodo) | `ZEYAROO_S3_SECRET_ACCESS_KEY` (Komodo) |
| Prod | `zeyaroo` | `ZEYAROO_PROD_S3_ACCESS_KEY_ID` (Komodo) | `ZEYAROO_PROD_S3_SECRET_ACCESS_KEY` (Komodo) |

## Steps

### 1. Authorize

```bash
curl -u "$S3_ACCESS_KEY_ID:$S3_SECRET_ACCESS_KEY" \
  "https://api.backblazeb2.com/b2api/v3/b2_authorize_account"
```

Note the `apiUrl`, `authorizationToken`, `accountId`, and `bucketId` from the response (the bucket ID is returned directly when the key is scoped to a single bucket).

### 2. Get bucket ID

Skip this step if the bucket ID was returned in step 1 (scoped keys include it). Otherwise:

```bash
curl \
  -H "Authorization: <authorizationToken>" \
  "<apiUrl>/b2api/v3/b2_list_buckets?accountId=<accountId>&bucketName=<bucket>"
```

### 3. Apply CORS rules

```bash
curl -X POST \
  -H "Authorization: <authorizationToken>" \
  -H "Content-Type: application/json" \
  "<apiUrl>/b2api/v3/b2_update_bucket" \
  -d '{
    "accountId": "<accountId>",
    "bucketId": "<bucketId>",
    "corsRules": [
      {
        "corsRuleName": "allowBrowserUploads",
        "allowedOrigins": [
          "http://localhost",
          "https://localhost",
          "http://localhost:5173",
          "http://localhost:4173",
          "http://localhost:3000",
          "http://localhost:3001",
          "http://localhost:8080",
          "http://localhost:8000",
          "https://*.bytkloud.com",
          "https://*.zeyaroo.com",
          "https://zeyaroo.com"
        ],
        "allowedHeaders": [
          "authorization",
          "content-type",
          "content-length",
          "x-bz-file-name",
          "x-bz-content-sha1",
          "x-bz-part-number",
          "x-amz-content-sha256",
          "x-amz-date",
          "x-amz-security-token"
        ],
        "allowedOperations": ["b2_upload_file", "b2_upload_part", "b2_download_file_by_id", "b2_download_file_by_name", "s3_put", "s3_head", "s3_get"],
        "exposeHeaders": ["ETag", "Content-Disposition"],
        "maxAgeSeconds": 3600
      }
    ]
  }'
```

## Notes

- B2 does not support port wildcards (e.g. `localhost:*`) — enumerate ports explicitly.
- Wildcards are only allowed at the **start** of an origin (e.g. `https://*.zeyaroo.com` is valid).
- These rules cover both the B2 native upload API and the S3-compatible API (presigned PUT and GET URLs).
- `s3_get` / `b2_download_*` are required for browser `fetch()` downloads — the API server redirects (307) to a B2 presigned URL, and `fetch()` needs CORS on the final B2 response to read the body.
- For prod, remove the `localhost` entries if desired.

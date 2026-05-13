# Arcads external API — reference

Official Swagger UI: [https://external-api.arcads.ai/docs](https://external-api.arcads.ai/docs)  
OpenAPI JSON (machine-readable): `GET https://external-api.arcads.ai/docs-json`

## Base URL

`https://external-api.arcads.ai`

Override with env `ARCADS_BASE_URL` if Arcads provides a different host for your workspace.

## Authentication

The API uses **HTTP Basic** (`securitySchemes.basic` in OpenAPI).

- **Typical pattern:** use your **Arcads API key as the Basic auth username** and an **empty password** (this matches the common "Authorize" UX in Swagger: paste the key in the username field, leave password empty).
- **Env:** `ARCADS_API_KEY` — never commit it; load from `.env` locally.
- **401 / 403:** key missing, wrong, or lacks permission — run the setup flow in `SKILL.md` (editor-first `.env`).

### curl example

```bash
curl -sS -u "$ARCADS_API_KEY:" "https://external-api.arcads.ai/v1/products"
```

(`-u 'key:'` means password empty.)

## Model → route mapping

### Primary: unified v2 video endpoint

All video models are available through a single endpoint. Use this as the **primary route** for all video generation.

| Model | `model` value | Endpoint | Request body |
|-------|---------------|----------|-------------|
| **Sora 2** | `sora2` | `POST /v2/videos/generate` | `CreateVideoDto` |
| **Sora 2 Pro** | `sora2-pro` | `POST /v2/videos/generate` | `CreateVideoDto` |
| **Veo 3.1** | `veo31` | `POST /v2/videos/generate` | `CreateVideoDto` |
| **Kling 2.6** | `kling-2.6` | `POST /v2/videos/generate` | `CreateVideoDto` |
| **Kling 3.0** | `kling-3.0` | `POST /v2/videos/generate` | `CreateVideoDto` |
| **Grok Video** | `grok-video` | `POST /v2/videos/generate` | `CreateVideoDto` |
| **Seedance 2.0** | `seedance-2.0` | `POST /v2/videos/generate` | `CreateVideoDto` |

### Other endpoints (not on the v2 unified route)

| Model / flow | Endpoint | Request body |
|-------------|----------|-------------|
| **Nano Banana (image)** | `POST /v2/images/generate` | `CreateImageDto` — default `model`: `nano-banana-2`; optional `nano-banana` (Nano Banana Pro) |
| **B-roll** | `POST /v1/b-roll` | `CreateBRollDto` |
| **Scene** | `POST /v1/scene` | `CreateSceneDto` |

## Common payloads

See the **Swagger UI** (`/docs`) for full schema definitions. Below are key fields and patterns.

### `CreateVideoDto` (unified v2 endpoint)

```json
{
  "productId": "<uuid>",
  "prompt": "<string, ~200-500 words typical>",
  "model": "sora2" | "veo31" | "kling-3.0" | "sora2-pro" | ...,
  "duration": 4 | 8 | 12 | 16 | 20,              // optional for Veo (auto), required for Sora/Kling
  "resolution": "720p" | "1080p" | "4k",         // optional; Veo default: 720p
  "aspectRatio": "16:9" | "9:16" | "1:1",        // optional; depends on model
  "startFrame": "<presigned-url-or-file-path>",  // Veo ONLY (mutually exclusive w/ refImageAsBase64)
  "refImageAsBase64": "<base64-string>",          // Sora 2 Pro / Kling (style ref only, not face-preserving)
  "referenceImages": ["<file-path-1>", ...],      // Style refs; Veo supports but prefers startFrame
  "audioEnabled": true | false,                    // optional
  "projectId": "<uuid>"                           // optional; assign to project
}
```

**Key points:**
- `model` is **required**.
- `duration` is **required for Sora, Kling, Seedance** but **optional for Veo** (Veo auto-determines, ~8s typical).
- `startFrame` (Veo only): animates from exact image. **Mutually exclusive** with `refImageAsBase64`.
- `refImageAsBase64` (Sora 2 Pro, Kling): style/mood inspiration only. Does NOT preserve face/pose.

### `CreateImageDto` (Nano Banana)

```json
{
  "productId": "<uuid>",
  "prompt": "<string>",
  "model": "nano-banana-2" | "nano-banana" | "gpt-image" | ...,
  "aspectRatio": "1:1" | "9:16" | "16:9",
  "referenceImages": ["<file-path-1>", ...],      // optional
  "projectId": "<uuid>"                           // optional
}
```

### `CreateBRollDto`

```json
{
  "productId": "<uuid>",
  "prompt": "<string>",
  "duration": 5 | 10,
  "aspectRatio": "16:9" | "9:16" | "1:1",
  "referenceImages": ["<file-path-1>", ...],
  "projectId": "<uuid>"
}
```

### `CreateSceneDto`

```json
{
  "productId": "<uuid>",
  "prompt": "<string>",
  "aspectRatio": "16:9" | "9:16" | "1:1",
  "referenceImages": ["<file-path-1>", ...],
  "projectId": "<uuid>"
}
```

(No `duration` required.)

## Endpoints

### Products

- `GET /v1/products` — list all products (paginated).
- `POST /v1/products` — create new product.
- `GET /v1/products/{productId}` — get product details.

### Folders & Projects

- `GET /v1/products/{productId}/folders` — list folders for product.
- `POST /v1/folders` — create folder.
- `GET /v1/projects` — list all projects.
- `POST /v1/projects` — create project.

### Assets & Generation

- `POST /v2/videos/generate` — generate video (all models).
- `POST /v2/images/generate` — generate image (Nano Banana, etc.).
- `POST /v1/b-roll` — generate b-roll.
- `POST /v1/scene` — generate scene.
- `GET /v1/assets/{id}` — poll asset status.
- `POST /v1/assets/add-to-project` — assign asset to project.

### File Upload

- `POST /v1/file-upload/get-presigned-url` — request presigned S3 URL for upload.
  - Body: `{ "fileType": "image" | "video" | "audio" }`
  - Response: `{ "presignedUrl": "...", "filePath": "..." }`
  - Follow up with `PUT <presignedUrl>` to upload file.

## Response structure

All responses are JSON.

### Success (200, 201)

```json
{
  "id": "<uuid>",
  "status": "generated" | "pending" | "failed",
  "url": "<presigned-s3-url>",
  "thumbnailUrl": "<presigned-s3-url>",
  "createdAt": "2026-04-09T12:00:00Z",
  "creditsCarged": 0.9,
  ...
}
```

### Error (4xx, 5xx)

```json
{
  "statusCode": 400,
  "message": "Invalid request",
  "error": "Bad Request"
}
```

## Polling pattern

```
1. POST /v2/videos/generate → asset { id, status: "pending" }
2. wait 5–30s
3. GET /v1/assets/{id} → repeat until status = "generated" | "failed"
4. Download from response.url (presigned URL expires in ~24h)
```

Typical generation times:
- Nano Banana image: ~35s
- Scene: ~75s
- Veo 3.1: ~4 min
- B-roll: ~5 min

## Rate limits & quotas

Check the Swagger UI or your Arcads dashboard for account-specific limits.

## Errors & troubleshooting

### 401 Unauthorized

- API key missing, wrong, or not in Basic auth header.
- Verify: `curl -u "$ARCADS_API_KEY:" https://external-api.arcads.ai/v1/products`

### 403 Forbidden

- API key valid but lacks permission for this resource.
- Check your Arcads dashboard permissions.

### 400 Bad Request

- Invalid payload (missing required field, wrong type, etc.).
- Check schema in Swagger UI.

### 429 Too Many Requests

- Rate limited. Retry after delay (check `Retry-After` header).

### Asset status: "failed"

- Generation error (invalid prompt, unsupported combination, etc.).
- Check response for `error` / `message` fields.
- Retry with adjusted prompt or parameters.

## Further reading

- **Official Swagger UI:** https://external-api.arcads.ai/docs
- **OpenAPI JSON:** https://external-api.arcads.ai/docs-json

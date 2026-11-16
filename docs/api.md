# REST API

Base URL: `http://127.0.0.1:8000` locally (`MONTAJ_HOST`, `MONTAJ_PORT`). Product endpoints live under `/v1`, take and return JSON (uploads are multipart) and require an API key. FastAPI serves the generated OpenAPI document at `/openapi.json` with UIs at `/docs` and `/redoc`.

Contents: [Authentication](#authentication-and-scopes), [Errors](#errors), [Idempotency](#idempotency), [Rate limits and quotas](#rate-limits-and-quotas), [Pagination](#pagination), [Request limits](#request-limits-and-headers), [Endpoints](#endpoints), [Jobs](#jobs), [Assets](#asset-object), [Revisions](#revisions), [Webhooks](#webhooks), [Usage](#usage).

## Authentication and scopes

Send the key as a bearer token:

```text
Authorization: Bearer mj_live_abcdefghijkl_<52 characters>
```

Key format: `mj_<env>_<prefix>_<secret>`. `<env>` is `MONTAJ_KEY_ENV` at creation time (default `live`, 1 to 8 lowercase letters or digits), `<prefix>` is 12 characters used for lookup, `<secret>` is 256 random bits in lowercase base32 (52 characters). The server stores only HMAC-SHA256 of `prefix.secret` with the server pepper (`MONTAJ_KEY_PEPPER`). `last_used_at` is updated at most once a minute.

Scopes are ranked `admin > write > read`; a key satisfies every scope at or below its own.

| Scope | Allows |
|---|---|
| `read` | GET endpoints, `POST /v1/renders/estimate` |
| `write` | creating and deleting assets, jobs, music tracks and webhooks; cancelling and revising jobs |
| `admin` | `/v1/api-keys` |

Missing key: 401 `unauthorized`. A header that is not `Bearer <key>`: 401 `invalid_authorization_header`. Unknown, malformed or revoked key: 401 `invalid_api_key`. All 401 responses carry `WWW-Authenticate: Bearer`. A valid key without the scope: 403 `insufficient_scope` with `details.required`.

Every key belongs to one organisation (`org_id`). All data is filtered by it; another organisation's ids answer 404.

The first key is created on the server with `montaj-api admin create-key` (see [quickstart.md](quickstart.md#3-create-the-first-api-key)).

## Errors

Every error, including unknown routes, wrong methods, validation errors and unhandled exceptions, uses one envelope:

```json
{
  "error": {
    "code": "validation_error",
    "message": "request validation failed",
    "details": [{"loc": ["body", "prompt"], "msg": "Field required", "type": "missing"}],
    "request_id": "req_5f2c0e9a1b3d4c6e7f8a9b0c"
  }
}
```

Validation `details` list `loc`, `msg` and `type` only; the submitted input is never echoed back. A 500 is logged with its traceback; the client gets `internal_error` with no details.

Error codes returned by the API:

| HTTP | Code | When |
|---|---|---|
| 400 | `bad_request` | generic malformed request |
| 400 | `invalid_cursor` | `cursor` is not an id from this list |
| 400 | `invalid_idempotency_key` | `Idempotency-Key` is not 1 to 255 printable ASCII characters |
| 401 | `unauthorized`, `invalid_authorization_header`, `invalid_api_key` | see [Authentication](#authentication-and-scopes) |
| 403 | `insufficient_scope` | the key lacks the scope |
| 403 | `invalid_signature` | a signed download link is invalid or expired |
| 404 | `not_found` | unknown id (or another organisation's), unknown route, `/metrics` when disabled |
| 404 | `asset_not_found` | a request names asset ids that do not exist; `details.missing` lists them |
| 405 | `method_not_allowed` | wrong HTTP method |
| 409 | `job_not_cancellable` | the job already finished |
| 409 | `job_not_revisable` | the parent job did not succeed (or has no output image) |
| 409 | `webhook_limit_exceeded` | the organisation already has `MONTAJ_MAX_WEBHOOKS_PER_ORG` (default 10) active webhooks (`details.limit`) |
| 409 | `idempotency_key_reused` | the key was used with a different request |
| 409 | `idempotency_request_in_progress` | the first request with this key is still running (`Retry-After: 1`) |
| 413 | `payload_too_large` | body or file over the limit |
| 415 | `unsupported_media_type` | empty file, unknown format, or the content does not decode |
| 422 | `validation_error` | the body or query failed validation |
| 422 | `invalid_asset_kind` | an asset has the wrong kind (`details.asset_ids`, or `clip_asset_ids` and `audio_asset_ids` for timelines) |
| 422 | `invalid_operations` | image operations failed validation (`details` lists each problem) |
| 422 | `media_limits_exceeded` | duration or resolution over the limits |
| 422 | `url_not_allowed` | URL import or webhook URL refused by the SSRF guard |
| 422 | `import_failed` | the remote server answered with a non-2xx status (`details.status_code`) |
| 422 | `import_timeout`, `import_too_many_redirects`, `import_fetch_failed` | the URL fetch failed |
| 422 | `revision_unsupported` | a timeline revision of an image edit, or a prompt revision of a render |
| 422 | `invalid_period` | usage period invalid or longer than 366 days |
| 429 | `rate_limited` | rate limit hit (`Retry-After`, `details.retry_after_seconds`) |
| 429 | `job_quota_exceeded` | too many queued or running jobs for the organisation (`Retry-After: 5`) |
| 500 | `internal_error` | unhandled server error |
| 503 | `not_ready` | `/ready` found a failing dependency |

Job failures are not HTTP errors: they are stored in the job's `error` field (see [Job errors](#job-errors)).

## Idempotency

Any authenticated `POST` may carry `Idempotency-Key: <1 to 255 printable ASCII characters>`. Records are scoped to the API key that sent the request, not to its organisation: another key of the same organisation, whatever its scopes, never receives a stored response.

- The first request runs normally. Its response is stored with a hash of the request (method, path, query, media type and body; multipart boundaries are normalised so a retried upload matches) for `MONTAJ_IDEMPOTENCY_TTL_HOURS` (default 24 h).
- A repeat with the same key and the same request returns the stored status and body with `Idempotent-Replayed: true`. No second job or asset is created.
- The same key with a different request: 409 `idempotency_key_reused`.
- A repeat while the first is still being processed: 409 `idempotency_request_in_progress` with `Retry-After: 1`. An in-progress record older than 15 minutes (left behind by a crashed process) is reclaimed, and the request runs again.
- Responses with status 5xx, 408, 409, 425 or 429, and response bodies over 1 MiB, are not stored, so such a request can be retried with the same key.
- `POST /v1/api-keys` and `POST /v1/webhooks` bypass idempotency entirely, because their responses carry a one-time secret that is never persisted. The header is ignored there, so a retry can create a second key or webhook; revoke or delete the extra one.

The SDK sends a fresh key on every POST and reuses it across its own retries.

## Rate limits and quotas

- **Rate limit:** a token bucket per API key (per client IP for unauthenticated requests). Refill `MONTAJ_RATE_LIMIT_PER_MINUTE` (default 120 per minute), capacity `MONTAJ_RATE_LIMIT_BURST` (default 60). Over the limit: 429 `rate_limited` with `Retry-After` (whole seconds) and `details.retry_after_seconds`. `/health`, `/ready` and `/metrics` are exempt.
- **The buckets are in process memory**, so each API instance counts separately: with N instances the effective limit is N times the setting.
- **Job quota:** at most `MONTAJ_MAX_CONCURRENT_JOBS_PER_ORG` (default 4) queued or running jobs per organisation. Creating another job returns 429 `job_quota_exceeded` with `details.limit`, `details.active` and `Retry-After: 5`.

## Pagination

`GET /v1/assets` and `GET /v1/jobs` return newest first:

```json
{"object": "list", "data": [ ... ], "has_more": true, "next_cursor": "job_..."}
```

Pass `next_cursor` as `cursor` to get the next page. `limit` is 1 to 200 (default 50). An unknown cursor is 400 `invalid_cursor`.

Other lists return `{"object": "list", "data": [...]}` without cursors: `GET /v1/music` (`limit` 1 to 500, default 100), `GET /v1/webhooks/{id}/deliveries` (`limit` 1 to 200, default 50), `GET /v1/webhooks` and `GET /v1/api-keys` (all rows).

## Request limits and headers

| Limit | Default | Setting |
|---|---|---|
| Multipart upload body | 512 MiB file + 1 MiB framing | `MONTAJ_MAX_UPLOAD_BYTES` |
| Any other request body | 2 MiB | `MONTAJ_MAX_JSON_BODY_BYTES` |
| Video or audio duration | 900 s | `MONTAJ_MAX_MEDIA_DURATION_SECONDS` |
| Longest side of an image or video | 8192 px | `MONTAJ_MAX_MEDIA_DIMENSION` |

- `X-Request-ID`: sent back on every response. A client value matching `[A-Za-z0-9._-]{1,128}` is kept; otherwise the server generates `req_<24 hex>`. It also appears in the error envelope and the JSON logs.
- Every response carries `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Cross-Origin-Opener-Policy: same-origin` and (outside the docs pages) `Content-Security-Policy: default-src 'none'; frame-ancestors 'none'`. JSON responses add `Cache-Control: no-store`; HTTPS requests add `Strict-Transport-Security`.
- CORS is off unless `MONTAJ_CORS_ORIGINS` lists origins (methods GET, POST, DELETE; headers `Authorization`, `Content-Type`, `Idempotency-Key`, `X-Request-ID`).

## Endpoints

| Method and path | Scope | Success |
|---|---|---|
| `GET /health` | none | 200 |
| `GET /ready` | none | 200 or 503 |
| `GET /metrics` | none, or the metrics token when `MONTAJ_METRICS_TOKEN` is set | 200 (Prometheus text) |
| `POST /v1/api-keys` | admin | 201 |
| `GET /v1/api-keys` | admin | 200 |
| `DELETE /v1/api-keys/{key_id}` | admin | 200 |
| `POST /v1/assets` | write | 201 |
| `POST /v1/assets/import` | write | 201 |
| `GET /v1/assets` | read | 200 |
| `GET /v1/assets/{asset_id}` | read | 200 |
| `DELETE /v1/assets/{asset_id}` | write | 204 |
| `GET /v1/assets/{asset_id}/content` | read | 307 |
| `GET /v1/assets/{asset_id}/file` | signed URL | 200 |
| `POST /v1/montages` | write | 202 |
| `POST /v1/videos/edits` | write | 202 |
| `POST /v1/images/edits` | write | 202 |
| `POST /v1/renders` | write | 202 |
| `POST /v1/renders/estimate` | read | 200 |
| `GET /v1/jobs` | read | 200 |
| `GET /v1/jobs/{job_id}` | read | 200 |
| `POST /v1/jobs/{job_id}/cancel` | write | 200 or 202 |
| `POST /v1/jobs/{job_id}/revisions` | write | 202 |
| `GET /v1/music` | read | 200 |
| `POST /v1/music` | write | 201 |
| `GET /v1/music/{track_id}` | read | 200 |
| `POST /v1/webhooks` | write | 201 |
| `GET /v1/webhooks` | read | 200 |
| `GET /v1/webhooks/{webhook_id}` | read | 200 |
| `DELETE /v1/webhooks/{webhook_id}` | write | 204 |
| `GET /v1/webhooks/{webhook_id}/deliveries` | read | 200 |
| `GET /v1/usage` | read | 200 |

All examples below assume `export MONTAJ=http://127.0.0.1:8000` and `export KEY=mj_live_...`.

### Health

`GET /health` returns `{"status": "ok"}` without touching the database. `GET /ready` checks the database, a storage write and that `ffmpeg` and `ffprobe` run (the FFmpeg result is cached for 60 s):

```json
{"status": "ready", "checks": {"database": "ok", "storage": "ok", "ffmpeg": "ok"}}
```

The `/ready` result is cached for 5 s, so a flood of probes cannot hammer the database and storage. On failure: 503 `not_ready` with `details.checks`, for example `"ffmpeg": "error: ffmpeg/ffprobe not found"`. `GET /metrics` serves Prometheus metrics (404 when `MONTAJ_METRICS_ENABLED=false`). It takes no API key. When `MONTAJ_METRICS_TOKEN` is set, the caller must send `Authorization: Bearer <metrics token>`, else 401 `unauthorized` in the standard error envelope with `WWW-Authenticate: Bearer`; unset, it is open, which suits a private network only (production logs a warning at startup). See [deployment.md](deployment.md#observability).

### API keys

**`POST /v1/api-keys`** (admin). Body:

| Field | Type | Constraints |
|---|---|---|
| `name` | string | required, 1 to 200 characters |
| `scopes` | list of `read`, `write`, `admin` | at least one; default `["read", "write"]` |

Response 201: the key object plus `secret` (the full `mj_...` token, shown only here). The request ignores `Idempotency-Key` (see [Idempotency](#idempotency)), so a retried call creates another key.

```json
{"id": "key_...", "object": "api_key", "org_id": "org_...", "name": "ci", "prefix": "mj_live_abcdefghijkl",
 "scopes": ["read", "write"], "created_at": "2026-11-14T10:00:00Z", "last_used_at": null, "revoked_at": null,
 "secret": "mj_live_abcdefghijkl_..."}
```

```bash
curl -s -X POST $MONTAJ/v1/api-keys -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" -d '{"name": "ci", "scopes": ["read", "write"]}'
```

**`GET /v1/api-keys`** lists the organisation's keys (without secrets). **`DELETE /v1/api-keys/{key_id}`** revokes a key and returns it with `revoked_at` set; it stops authenticating immediately.

```bash
curl -s -X DELETE $MONTAJ/v1/api-keys/key_... -H "Authorization: Bearer $KEY"
```

### Assets

**`POST /v1/assets`** (write): multipart upload of one image, video or audio file.

| Form field | Constraints |
|---|---|
| `file` | required; at most `MONTAJ_MAX_UPLOAD_BYTES` |
| `filename` | optional, at most 255 characters; defaults to the uploaded file name |

Every file is checked the same way: not empty, the format is sniffed from the bytes (the client's content type and extension are ignored), `ffprobe` reads its headers, duration and resolution must be within the limits (checked from the probe before anything is decoded, so a small file declaring a huge picture is refused cheaply), FFmpeg then decodes the first frame (or first second of audio), and the sniffed family must match the decoded kind. A SHA-256 content hash is stored. Job outputs skip the configurable limits but are held to a hard ceiling of 16384 px per side and 6 h.

Sniffed formats: JPEG, PNG, GIF, WebP, TIFF, BMP, HEIC, AVIF (images); MP4, MOV, WebM, Matroska, AVI, MPEG-TS (video); WAV, MP3, AAC, M4A, FLAC, Ogg (audio). The file must also decode with your FFmpeg build. The asset's `kind` comes from ffprobe (an MP4 with only an audio stream is an `audio` asset).

Errors: 413 `payload_too_large`, 415 `unsupported_media_type`, 422 `media_limits_exceeded`.

```bash
curl -s -X POST $MONTAJ/v1/assets -H "Authorization: Bearer $KEY" -F "file=@beach.jpg"
```

**`POST /v1/assets/import`** (write): the server fetches a public URL.

| Field | Constraints |
|---|---|
| `url` | required, 1 to 2048 characters, `http` or `https`, no credentials |
| `filename` | optional, at most 255 characters; defaults to the URL's last path segment |

The SSRF guard resolves the host once and refuses the URL if any address is private, loopback, link-local, multicast, reserved or otherwise not globally routable (IPv4 and IPv6, including IPv4-mapped, 6to4, Teredo and NAT64 forms). The connection goes to the vetted address. Redirects are followed by hand, each hop re-checked, up to `MONTAJ_IMPORT_MAX_REDIRECTS` (default 3). Size is capped at `MONTAJ_MAX_UPLOAD_BYTES` and time at `MONTAJ_IMPORT_TIMEOUT_SECONDS` (default 60). `MONTAJ_ALLOW_PRIVATE_URLS=true` skips only the address check (for tests and local development).

Errors: 422 `url_not_allowed`, `import_failed`, `import_timeout`, `import_too_many_redirects`, `import_fetch_failed`; 413 `payload_too_large`; plus the upload errors.

```bash
curl -s -X POST $MONTAJ/v1/assets/import -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" -d '{"url": "https://example.com/clip.mp4"}'
```

**`GET /v1/assets`** (read): query `kind` (`image`, `video`, `audio`), `limit`, `cursor`. Job outputs are assets too (`source: "job"`).

```bash
curl -s "$MONTAJ/v1/assets?kind=video&limit=20" -H "Authorization: Bearer $KEY"
```

**`GET /v1/assets/{asset_id}`** (read): the [asset object](#asset-object) with a freshly signed `download_url`.

**`DELETE /v1/assets/{asset_id}`** (write): 204. The asset disappears from the API at once; the worker deletes its bytes after `MONTAJ_ASSET_PURGE_AFTER_HOURS` (default 24). Job outputs that referenced it then show `download_url: null`.

**`GET /v1/assets/{asset_id}/content`** (read): 307 redirect to a fresh signed download URL.

```bash
curl -s -L -o out.mp4 $MONTAJ/v1/assets/ast_.../content -H "Authorization: Bearer $KEY"
```

**`GET /v1/assets/{asset_id}/file?o=&e=&s=`** (no key): serves the bytes of a local-storage signed URL. `o` is the org id, `e` the expiry (unix seconds), `s` the HMAC-SHA256 signature with `MONTAJ_URL_SIGNING_SECRET`. A bad or expired signature is 403 `invalid_signature`. You never build this URL yourself: it is the `download_url` the API hands out when storage is local. With S3 storage `download_url` is an S3 presigned GET instead.

### Montages

**`POST /v1/montages`** (write): photos and/or clips plus a prompt become a beat-synced video job.

| Field | Type | Constraints and default |
|---|---|---|
| `asset_ids` | list of ids | required, 1 to 200, no repeats; live `image` or `video` assets |
| `prompt` | string | required, 1 to 2000 characters |
| `music_asset_id` | id or null | an audio asset, or a video with sound; not also in `asset_ids`. Omit to pick a library track for the style, else generate a procedural bed |
| `aspect` | `16:9`, `9:16`, `1:1`, `4:5` | omit to use the aspect the prompt names (vertical, reel, square ...), else the style's; the heuristic director switches a 16:9 style to 9:16 when at least two thirds of the media is portrait |
| `style` | `cinematic`, `upbeat`, `calm`, `travel`, `celebration`, `product`, `minimal` | omit to infer from the prompt |
| `target_duration` | number | 3 to 600 seconds; omit for what the material sustains |
| `quality` | `draft`, `standard`, `high` | default `standard` |
| `seed` | integer | 0 to 2147483647; omitted means random, and the chosen seed is stored in the job's `input` |

Unknown fields are rejected. Response 202: the queued [job](#jobs). Errors: 404 `asset_not_found`, 422 `invalid_asset_kind`, 429 `job_quota_exceeded`.

With the heuristic director (no Anthropic key on the workers), the same input with the same seed produces the same Timeline; the Claude director is not deterministic, but its result can be reproduced by rendering the returned Timeline. Videos shorter than 0.5 s are ignored as shots.

```bash
curl -s -X POST $MONTAJ/v1/montages -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" -H "Idempotency-Key: trip-2026-1" \
  -d '{"asset_ids": ["ast_A", "ast_B", "ast_C"], "prompt": "upbeat summer trip",
       "music_asset_id": "ast_M", "aspect": "9:16", "target_duration": 20, "quality": "draft"}'
```

### Video edits

**`POST /v1/videos/edits`** (write): one or more clips (photos may be mixed in) plus a prompt. Same fields as montages, plus:

| Field | Type | Default |
|---|---|---|
| `keep_audio` | boolean | `true`: every clip keeps its own sound at full volume (music, if any, is ducked under it); `false` mutes the clips |

Differences from a montage: music is used only when `music_asset_id` is given (no library pick, no procedural bed); without music the director plans on a fixed 0.5 s virtual beat grid. Without `aspect`, the canvas follows an aspect named in the prompt, else the shape of the first video.

```bash
curl -s -X POST $MONTAJ/v1/videos/edits -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"asset_ids": ["ast_CLIP"], "prompt": "cut to the best 15 seconds, warm colours", "aspect": "9:16"}'
```

### Image edits

**`POST /v1/images/edits`** (write): exactly one of `operations` or `prompt`.

| Field | Constraints |
|---|---|
| `asset_id` | required; a live `image` asset |
| `operations` | 1 to 50 operation objects, applied in order |
| `prompt` | 1 to 2000 characters; the director turns it into operations (keyword mapping without an Anthropic key) |

Operations are validated at request time (422 `invalid_operations`) and stored normalised, with every default filled in. Each operation is an object with an `op` key:

| `op` | Fields (default) |
|---|---|
| `resize` | `width`, `height` (1 to 12000, at least one), `mode` `fit`/`fill`/`exact` (`fit`; `fill` and `exact` need both sides), `allow_upscale` (false) |
| `crop` | `left`, `top` (>= 0), `width`, `height` (> 0), `unit` `px`/`fraction` (`px`; fractions must lie within 0..1) |
| `smart_crop` | `aspect` as `"W:H"` (1 to 3 digits each). Largest window of that aspect containing all detected faces, else centred on the focus point |
| `rotate` | `degrees` -360 to 360, clockwise; `expand` (true); `fill` (`#000000`, transparent when the image has alpha) |
| `flip` | `direction` `horizontal`/`vertical` (required) |
| `auto_enhance` | `strength` 0..1 (0.6) |
| `adjust` | `brightness` -1..1 (0), `contrast` 0..3 (1), `saturation` 0..3 (1), `temperature` -1..1 (0), `sharpness` 0..3 (1) |
| `denoise` | `strength` 0..1 (0.5) |
| `color_preset` | `preset` `none`/`warm`/`cool`/`vivid`/`film`/`mono` (required), `intensity` 0..1 (1) |
| `text` | `text` 1 to 500 characters, `position` `top`/`center`/`bottom`/`top_left`/`top_right`/`bottom_left`/`bottom_right` (`bottom`), `size_scale` 0.2..5 (1), `color` (`#ffffff`), `stroke_color` (`#000000` or null) |
| `convert` | `format` `jpeg`/`png`/`webp` (required), `quality` 1..100 (90), `background` (`#ffffff`, used to flatten transparency for JPEG), `strip_metadata` (false) |
| `remove_background` | no fields. Needs the optional `rembg` extra (model `birefnet-general`); without it the job fails with `invalid_request` |

Canvas cap: a source over 144 megapixels is refused before it is decoded, and after every operation the image may not have grown past 12000 px per side or 144 megapixels (a `rotate` with `expand` is refused before the larger canvas is allocated). Either way the job fails with `invalid_request`.

The output format is the `convert` format, else the source format when it is JPEG, PNG or WebP, else PNG. EXIF orientation is applied to the pixels; other metadata is kept where the format allows.

```bash
curl -s -X POST $MONTAJ/v1/images/edits -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"asset_id": "ast_IMG", "operations": [{"op": "smart_crop", "aspect": "1:1"},
       {"op": "resize", "width": 1080}, {"op": "convert", "format": "webp", "quality": 85}]}'
```

### Renders

**`POST /v1/renders`** (write): render an explicit [Timeline](timeline.md). Body `{"timeline": {...}}`. Limits: at most 1000 clips and 600 s of video track. Every clip's asset must exist with a kind matching `clip.kind`; every audio clip's asset must be audio or a video with sound (422 `invalid_asset_kind` otherwise).

```bash
curl -s -X POST $MONTAJ/v1/renders -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" -d @render.json      # {"timeline": {...}}
```

**`POST /v1/renders/estimate`** (read): the same body, no job. Response 200:

```json
{"object": "render_estimate", "duration": 6.1, "frames": 183, "clips": 3, "segments": 3,
 "transitions": 1, "transition_types": ["slide_left"], "cuts": 1, "texts": 1, "audio_tracks": 1,
 "output": {"width": 1080, "height": 1920, "fps": 30, "video_codec": "h264", "quality": "standard"}}
```

```bash
curl -s -X POST $MONTAJ/v1/renders/estimate -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" -d @render.json
```

### Jobs

**`GET /v1/jobs`** (read): query `status` (`queued`, `running`, `succeeded`, `failed`, `cancelled`), `type` (for example `montage`), `limit`, `cursor`.

```bash
curl -s "$MONTAJ/v1/jobs?status=running" -H "Authorization: Bearer $KEY"
```

**`GET /v1/jobs/{job_id}`** (read): the [job object](#job-object). Asset objects inside `output` get freshly signed download URLs on every read.

```bash
curl -s $MONTAJ/v1/jobs/job_... -H "Authorization: Bearer $KEY"
```

**`POST /v1/jobs/{job_id}/cancel`** (write): a queued job is cancelled at once (200, status `cancelled`). A running job gets `cancel_requested: true` (202); the pipeline sees it within one heartbeat, any running FFmpeg process is killed, and the job ends `cancelled`. A cancel that arrives after the pipeline has finished is too late: the job ends `succeeded` and is billed as usual. A finished job: 409 `job_not_cancellable`.

```bash
curl -s -X POST $MONTAJ/v1/jobs/job_.../cancel -H "Authorization: Bearer $KEY"
```

**`POST /v1/jobs/{job_id}/revisions`** (write): see [Revisions](#revisions).

### Music

**`POST /v1/music`** (write): register an uploaded audio asset as a library track. Library tracks are what montages pick from when no music is given.

| Field | Constraints |
|---|---|
| `asset_id` | required; a live `audio` asset |
| `title` | required, 1 to 300 characters |
| `licence` | required, 1 to 5000 characters: licence text or identifier |
| `attribution` | optional, at most 1000 characters |
| `tags` | up to 32 strings of 1 to 50 characters; stored lower-case without duplicates. Style names and moods help selection |
| `tempo` | 20 to 300 BPM; omit to have it measured from the audio when the track is first considered |
| `energy` | 0 to 1; treated as 0.5 when omitted |

Response 201:

```json
{"id": "mus_...", "object": "music_track", "title": "Sunrise", "licence": "CC-BY-4.0",
 "attribution": "Artist Name", "tags": ["upbeat", "travel"], "tempo": 118.0, "energy": 0.7,
 "duration": 95.4, "asset": {"id": "ast_...", "object": "asset", "...": "..."}, "created_at": "..."}
```

```bash
curl -s -X POST $MONTAJ/v1/music -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"asset_id": "ast_AUDIO", "title": "Sunrise", "licence": "CC-BY-4.0", "attribution": "Artist Name",
       "tags": ["upbeat", "travel"], "tempo": 118, "energy": 0.7}'
```

**`GET /v1/music`** (read): query `tag` (at most 50 characters), `limit` (1 to 500, default 100), newest first. Tracks whose audio asset was deleted are hidden. Both filters run before the limit, so `limit` counts matching tracks. **`GET /v1/music/{track_id}`** (read): one track. There is no delete endpoint; deleting the audio asset hides the track.

```bash
curl -s "$MONTAJ/v1/music?tag=upbeat" -H "Authorization: Bearer $KEY"
```

### Webhooks

See [Webhooks](#webhooks) below for delivery and signatures.

**`POST /v1/webhooks`** (write). An organisation has at most `MONTAJ_MAX_WEBHOOKS_PER_ORG` (default 10) active webhooks; one more is 409 `webhook_limit_exceeded` (delete one first). The request ignores `Idempotency-Key` (see [Idempotency](#idempotency)).

| Field | Constraints |
|---|---|
| `url` | required, 1 to 2048 characters; must pass the SSRF guard |
| `events` | non-empty subset of `job.succeeded`, `job.failed`, `job.cancelled`; default all three |
| `description` | optional, at most 500 characters |

Response 201: `{"id": "whk_...", "object": "webhook", "url", "events", "description", "created_at", "secret": "whsec_<64 hex>"}`. The secret is shown only here.

```bash
curl -s -X POST $MONTAJ/v1/webhooks -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"url": "https://hooks.example.com/montaj", "events": ["job.succeeded", "job.failed"]}'
```

**`GET /v1/webhooks`**, **`GET /v1/webhooks/{id}`**: active webhooks (no secret). **`DELETE /v1/webhooks/{id}`**: 204; pending deliveries to it are then marked failed. **`GET /v1/webhooks/{id}/deliveries`** (`limit` 1 to 200, default 50): recent deliveries with their attempt log.

```bash
curl -s $MONTAJ/v1/webhooks/whk_.../deliveries -H "Authorization: Bearer $KEY"
```

### Usage

**`GET /v1/usage`** (read): query `start`, `end` (ISO-8601; a value without a time zone is UTC). Default: the current UTC calendar month. `end` must be after `start` and the period at most 366 days.

```bash
curl -s "$MONTAJ/v1/usage?start=2026-11-01T00:00:00Z&end=2026-12-01T00:00:00Z" -H "Authorization: Bearer $KEY"
```

See [Usage](#usage) for the response.

## Jobs

### Job object

```json
{
  "id": "job_0a1b2c3d4e5f6a7b8c9d0e1f",
  "object": "job",
  "type": "montage",
  "status": "succeeded",
  "progress": 1.0,
  "stage": "finalise",
  "input": {"asset_ids": ["ast_..."], "prompt": "upbeat summer trip", "music_asset_id": "ast_...",
            "aspect": "9:16", "style": null, "target_duration": 20.0, "quality": "draft", "seed": 1234567},
  "output": {
    "video": {"id": "ast_...", "object": "asset", "kind": "video", "download_url": "...", "...": "..."},
    "poster": {"id": "ast_...", "object": "asset", "kind": "image", "...": "..."},
    "timeline": {"version": "1", "seed": 1234567, "output": {"...": "..."}, "clips": ["..."]},
    "licence": {"source": "user", "track_id": "ast_...", "title": null,
                "licence": "Supplied by the user; the uploader is responsible for the rights to this track",
                "attribution": null},
    "director": {"source": "heuristic", "notes": ["..."], "input_tokens": 0, "output_tokens": 0},
    "music": {"asset": {"id": "ast_...", "object": "asset", "...": "..."}, "generated": false, "tempo": 120.0},
    "segments": {"rendered": 7, "reused": 0},
    "duration": 20.0,
    "seed": 1234567
  },
  "error": null,
  "attempts": 1,
  "max_attempts": 3,
  "cancel_requested": false,
  "created_at": "2026-11-14T10:00:00Z",
  "updated_at": "2026-11-14T10:00:41Z",
  "started_at": "2026-11-14T10:00:01Z",
  "finished_at": "2026-11-14T10:00:41Z"
}
```

`type` is `montage`, `video_edit`, `image_edit` or `render`. `input` is the request as stored (with the seed filled in and, for revisions, `parent_job_id` and `revision_prompt`).

Output of the video types (`montage`, `video_edit`, `render`):

| Field | Meaning |
|---|---|
| `video`, `poster` | asset objects of the MP4 and a JPEG poster frame (taken from the middle of the first clip) |
| `timeline` | the rendered Timeline JSON; edit it and send it back as a revision |
| `licence` | the music's [licence record](styles-and-music.md#the-licence-record), or null when there is no music |
| `director` | `source` (`claude`, `heuristic`, `heuristic_fallback`, or `none` for explicit renders), `notes`, token counts |
| `music` | `{"asset", "generated", "tempo"}`: the music asset (a generated bed is stored as `music.wav`), or null |
| `segments` | how many clip segments were rendered and how many were reused from the cache |
| `duration`, `seed` | output length in seconds and the seed |

Output of `image_edit`: `{"image": <asset>, "operations": [...], "director": {...}, "notes": [...]}`.

### Lifecycle

```text
queued --claim--> running --> succeeded
   |                 |------> failed
   |                 |------> cancelled  (cancel requested while the pipeline runs)
   |                 '--heartbeat stale--> queued again (or failed: heartbeat_lost)
   '--cancel--> cancelled
```

- A worker claims the oldest queued job, sets `running`, increments `attempts` and heartbeats every `MONTAJ_JOB_HEARTBEAT_SECONDS` (5).
- `progress` only moves forward. Stages: `analyse`, `music`, `direct`, `conform`, `render`, `finalise` for montages and video edits; `render`, `finalise` for renders; `direct` (prompt only), `render`, `finalise` for image edits.
- If a worker dies, its job's heartbeat goes stale after `MONTAJ_JOB_STALE_AFTER_SECONDS` (60) and the job is queued again, until `max_attempts` (`MONTAJ_JOB_MAX_ATTEMPTS`, 3) claims are used; then it fails with `heartbeat_lost`.
- A pipeline error fails the job immediately; it is not retried.
- Every finished job fires the matching webhook event and records usage.

### Job errors

`error` is `{"code", "message", "details"}`:

| Code | Meaning |
|---|---|
| `invalid_request` | the input cannot be processed (unknown asset in the pipeline, audio given as a shot, a prompt that maps to no image operation, a missing optional dependency, an image over the canvas cap) |
| `no_usable_assets` | no image, and no video of at least 0.5 s |
| `music_too_short` | the track is shorter than 2 s or has fewer beats than one bar, or the plan does not fit in it |
| `director_failed` | no valid edit plan could be produced (`details.errors`) |
| `render_failed` | rendering failed; for FFmpeg errors the message is the generic `media processing failed` and `details` is `{"returncode": n}` |
| `ffmpeg_failed` | an FFmpeg error outside the core pipeline, with the same generic message and details |
| `asset_not_found` | an input asset was deleted before the job ran |
| `output_invalid` | an output file could not be stored |
| `cancelled` | cancelled while running |
| `heartbeat_lost` | the worker stopped responding on the last allowed attempt |
| `pipeline_error` | an unexpected exception; the message is generic (`the job failed with an internal error`) |

The full detail of `render_failed`, `ffmpeg_failed` and `pipeline_error` (command, stderr tail, traceback) is written to the worker logs with the job id, never to the job.

## Asset object

```json
{
  "id": "ast_1a2b3c4d5e6f7a8b9c0d1e2f",
  "object": "asset",
  "kind": "video",
  "mime_type": "video/mp4",
  "filename": "clip_1.mp4",
  "size_bytes": 482113,
  "content_hash": "<sha256 hex>",
  "width": 640,
  "height": 360,
  "duration": 3.0,
  "fps": 30.0,
  "has_audio": true,
  "rotation": 0,
  "is_hdr": false,
  "codec": "h264",
  "source": "upload",
  "source_url": null,
  "job_id": null,
  "created_at": "2026-11-14T10:00:00Z",
  "download_url": "http://127.0.0.1:8000/v1/assets/ast_.../file?o=org_...&e=1763115300&s=...",
  "download_url_expires_at": "2026-11-14T10:15:00Z"
}
```

`source` is `upload`, `url` (with `source_url`) or `job` (with `job_id`). `download_url` is valid for `MONTAJ_SIGNED_URL_TTL_SECONDS` (default 900). It is built from `MONTAJ_PUBLIC_BASE_URL` for local storage, so set that to the address clients use.

## Revisions

`POST /v1/jobs/{job_id}/revisions` creates a new job from a **succeeded** `montage`, `video_edit`, `image_edit` or `render` job. Give exactly one of:

| Field | Result |
|---|---|
| `timeline` | a full patched Timeline (same limits as `/v1/renders`). Queues a `render` job. If the patch has no `licence`, its audio uses only the parent's audio assets and the parent had a licence, the parent's licence is kept. |
| `prompt` | 1 to 2000 characters. Montages and video edits re-run with the parent's input, music (a generated bed is reused as its stored asset, with its licence) and seed, plus the revision prompt; successive revision prompts are joined (the last 4000 characters are kept). Image edits apply the prompt to the parent's output image. |

The new job's `input.parent_job_id` names the parent. Clips whose pixels did not change are reused from the segment cache (`output.segments.reused`). Errors: 404 unknown job, 409 `job_not_revisable`, 422 `revision_unsupported` (a timeline for an image edit, or a prompt for a render).

```bash
curl -s -X POST $MONTAJ/v1/jobs/job_.../revisions -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" -d '{"prompt": "slower cuts, no title"}'
```

## Webhooks

Events: `job.succeeded`, `job.failed`, `job.cancelled`. Deliveries are queued in the same transaction as the job's state change and sent by a separate delivery loop in the **worker** (not inline in the job loop, so a slow endpoint never delays job processing). Webhooks therefore need a running worker. Retries survive restarts.

Request: `POST <your url>`, `Content-Type: application/json`, with these headers:

| Header | Value |
|---|---|
| `X-Montaj-Event` | the event type |
| `X-Montaj-Event-Id` | `evt_...` (the same for every webhook that receives the event) |
| `X-Montaj-Delivery` | `whd_...` (one per webhook and event) |
| `X-Montaj-Timestamp` | unix seconds when this attempt was sent |
| `X-Montaj-Signature` | `v1=<hex HMAC-SHA256(secret, "<timestamp>.<raw body>")>` |

Body (compact JSON, sorted keys):

```json
{"created_at": "2026-11-14T10:00:41Z", "data": {"job": {"id": "job_...", "object": "job", "status": "succeeded", "...": "..."}},
 "id": "evt_...", "object": "event", "type": "job.succeeded"}
```

`data.job` is the job object at the time of the event, without signed download URLs: asset objects inside `output` keep their ids but have no `download_url` or `download_url_expires_at` (a retried delivery carries the same body, and URLs would have expired). Fetch `GET /v1/jobs/{id}` for fresh URLs.

Delivery rules: a 2xx answer is success. Anything else, including redirects (never followed), timeouts (`MONTAJ_WEBHOOK_TIMEOUT_SECONDS`, 10) and refused addresses, is retried after `base * 2^(attempt - 1)` seconds capped at the maximum (`MONTAJ_WEBHOOK_BACKOFF_BASE_SECONDS` 30, `MONTAJ_WEBHOOK_BACKOFF_MAX_SECONDS` 3600), up to `MONTAJ_WEBHOOK_MAX_ATTEMPTS` (8) attempts. The SSRF guard runs at registration and on every attempt. Deliveries are listed at `GET /v1/webhooks/{id}/deliveries` with `status` (`pending`, `delivered`, `failed`), `attempts`, `next_attempt_at`, `last_status_code`, `last_error` and `attempt_log`.

### Verifying a delivery (Python)

Verify against the raw body bytes, before parsing the JSON, and reject old timestamps to stop replays. With the SDK:

```python
from montaj import verify_webhook

ok = verify_webhook(secret, request.headers, raw_body, tolerance_seconds=300)
```

With the standard library only:

```python
import hashlib
import hmac
import time


def verify(
    secret: str, timestamp: str, signature: str, raw_body: bytes, tolerance: int = 300
) -> bool:
    try:
        ts = int(timestamp)
    except ValueError:
        return False
    if abs(time.time() - ts) > tolerance:
        return False
    mac = hmac.new(secret.encode(), f"{ts}.".encode() + raw_body, hashlib.sha256)
    return hmac.compare_digest("v1=" + mac.hexdigest(), signature)


# timestamp = headers["X-Montaj-Timestamp"], signature = headers["X-Montaj-Signature"]
```

Use `X-Montaj-Event-Id` to ignore duplicates: a delivery can arrive more than once if your 2xx answer was lost.

## Usage

Pipelines meter usage; it is written when the job finishes. A succeeded job's events are billable; a failed or cancelled job's events are recorded with `billable: false` (zero cost). Each finished job also adds one `jobs` event.

| Metric | Unit |
|---|---|
| `output_seconds` | seconds of rendered video (video jobs) |
| `images_processed` | edited images |
| `director_input_tokens`, `director_output_tokens` | Claude tokens (all rounds, cache reads and writes included in input) |
| `jobs` | finished jobs |

`GET /v1/usage` returns billable totals:

```json
{"object": "usage", "period": {"start": "2026-11-01T00:00:00Z", "end": "2026-12-01T00:00:00Z"},
 "metrics": [{"metric": "output_seconds", "quantity": 184.5, "events": 9},
             {"metric": "jobs", "quantity": 11.0, "events": 11}]}
```

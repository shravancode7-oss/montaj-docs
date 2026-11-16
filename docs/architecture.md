# Architecture

## Components

```text
           clients
  montaj CLI / SDK      MCP client (Claude Code)
        |                   |
        |              montaj-mcp  (apps/mcp, uses the SDK)
        |                   |
        v                   v
   +-----------------------------+        +------------------+
   |  API  (montaj-api serve)    |        |  storage         |
   |  FastAPI, apps/api          |------->|  local disk / S3 |
   +-----------------------------+        +------------------+
        |  rows: assets, jobs, ...                ^
        v                                         |
   +-----------------------------+                |
   |  database = job queue       |                |
   |  SQLite (WAL) or Postgres   |                |
   +-----------------------------+                |
        ^  claim, heartbeat, finish               |
        |                                         |
   +-----------------------------+                |
   |  worker (montaj-worker)     |----------------+
   |  pipelines from montaj_core |---> ffmpeg / ffprobe subprocesses
   |  webhook loop, maintenance  |---> Anthropic API (Claude director, optional)
   +-----------------------------+---> webhook endpoints
```

There is no separate queue or cache service. The database holds the jobs, and the worker's local disk holds the work and cache folders.

## Code layout

A uv workspace (`pyproject.toml`, Python 3.12) with four packages:

| Package | Path | Role |
|---|---|---|
| `montaj-core` | `packages/core/src/montaj_core` | everything that touches media; no HTTP, no database |
| `montaj-api` | `apps/api/src/montaj_api` | REST API, worker, migrations, `montaj-api` and `montaj-worker` commands |
| `montaj` | `packages/sdk/src/montaj` | Python SDK and the `montaj` CLI |
| `montaj-mcp` | `apps/mcp/src/montaj_mcp` | MCP server over the SDK |

`montaj_core`:

| Module | Contents |
|---|---|
| `analysis/` | `media.py` (probe, sharpness, exposure, focus, perceptual hash, candidate sub-clips), `faces.py` (Haar cascade faces), `dedupe.py` (near-duplicates), `beats.py` (beat grid), `opencv.py` |
| `director/` | `director.py` (interface, `make_director`), `claude.py`, `heuristic.py`, `styles.py`, `prompts.py`, `thumbnails.py` |
| `timeline/` | `schema.py` (Timeline), `plan.py` (EditPlan), `validate.py`, `conform.py` |
| `render/` | `probe.py`, `normalize.py` (per-clip segments), `segments.py` (segment cache), `assemble.py` (transitions, text, encode), `audio.py`, `text.py`, `ffmpeg.py` |
| `music/` | `provider.py` (style profiles, track scoring), `library.py` (folder manifest loader), `procedural.py` (generated bed) |
| `image/` | `ops.py` (operation models and implementations), `pipeline.py` |
| `pipelines/` | `montage.py`, `video_edit.py`, `image_edit.py`, `render.py`, `common.py`, `errors.py` |

`montaj_api` follows one folder per feature under `api/` (`api_keys`, `assets`, `health`, `images`, `jobs`, `montages`, `music`, `renders`, `usage`, `videos`, `webhooks`), each with a `*_router.py`, a `*_service.py` and a `__tests__/` folder. Features that own tables (`api_keys`, `assets`, `jobs`, `music`, `usage`, `webhooks`) add a `*_repository.py` and a `*_model.py`; `health` has a `health_repository.py` (the readiness database check) and no model; `images`, `videos`, `montages` and `renders` have neither and persist through the `jobs` and `assets` services. `jobs` also holds `jobs_pipelines.py`, which runs the core pipelines for the worker. Shared code is under `common/` (`config`, `interfaces`, `middleware`, `models`, `utils`). `server.py` builds the app and mounts every router; `main.py` runs uvicorn; `worker.py` is the worker; `cli.py` is the `montaj-api` command.

`scripts/check_structure.py` checks part of this layout: every `api/<feature>/` has a router and a `__tests__/` folder, routers do not import SQLAlchemy, repositories or models, only `settings.py` reads `os.environ`, test files live in `__tests__/` and there is no top-level `tests/` folder.

## Database

Two Alembic migrations live in `apps/api/migrations/versions/`. `0001_initial_schema.py` creates these tables: `orgs`, `api_keys`, `assets`, `jobs`, `usage_events`, `webhooks`, `webhook_deliveries`, `music_tracks`, `idempotency_keys`. `0002_idempotency_per_key.py` adds `key_id` to `idempotency_keys` and makes `(key_id, key)` unique, so idempotency records belong to the API key that sent the request (existing records are dropped; they are a 24 h replay cache). Every row except `orgs` carries an `org_id`; every query filters on it.

Ids are `<prefix>_<24 hex characters>`: `org`, `key`, `ast` (asset), `job`, `mus` (music track), `whk` (webhook), `whd` (webhook delivery), `evt` (event), `use` (usage event).

## Request path

A request passes through the middleware in this order:

1. **RequestId**: reads or creates `X-Request-ID`, logs the request.
2. **CatchAll**: turns unhandled exceptions into the error envelope.
3. **SecurityHeaders**.
4. **CORS**, only when `MONTAJ_CORS_ORIGINS` is set.
5. **BodyLimit**: 413 for oversized bodies.
6. **Auth**: resolves the bearer key to an org and scopes. The lookup is skipped (and the request treated as unauthenticated) while the client IP's rate-limit bucket is empty, so invalid-key floods do not reach the database.
7. **RateLimit**: token bucket per key, or per IP.
8. **Idempotency**: replays or records POST responses, per API key (`POST /v1/api-keys` and `POST /v1/webhooks` bypass it).
9. The router, then the service, then the repository.

Creating a job validates the request, checks that every asset exists in the org and has the right kind, checks the org's job quota, and inserts a `queued` job row. The API returns 202 at once. It never runs media work itself; uploads are only probed (duration and resolution limits are checked from the probe, before any decode) and decoded once to validate them.

## Job lifecycle

```text
queued --claim--> running --> succeeded | failed | cancelled
   ^                 |
   '--stale heartbeat' (until max_attempts, then failed: heartbeat_lost)
```

- **Claim.** Postgres: `SELECT ... FOR UPDATE SKIP LOCKED` on the oldest queued job. SQLite: an atomic `UPDATE ... WHERE status = 'queued' RETURNING`. Two workers never get the same job.
- **Run.** The worker creates a scratch folder under `MONTAJ_WORK_DIR` (input assets are read in place on local storage and downloaded into it on S3), runs the pipeline in a thread, heartbeats every `MONTAJ_JOB_HEARTBEAT_SECONDS`, and checks `cancel_requested` at each heartbeat. The pipeline polls for cancellation before and about every 0.5 s during each FFmpeg or ffprobe process and kills the process when the job is cancelled. A cancel that arrives after the pipeline finished is too late: the job ends `succeeded`.
- **Finish.** Outputs are stored as new assets (`source: "job"`), the job row is updated, usage events and webhook deliveries are written in the same transaction, the scratch folder is deleted (however the run ends, worker shutdown included) and the cache is trimmed to `MONTAJ_CACHE_MAX_BYTES`. FFmpeg and unexpected errors are logged in full with the job id; the job's `error` carries only a generic message (and the FFmpeg exit code).
- **Recover.** Any worker re-queues jobs whose heartbeat is older than `MONTAJ_JOB_STALE_AFTER_SECONDS`. After `MONTAJ_JOB_MAX_ATTEMPTS` claims the job fails with `heartbeat_lost`.

## The montage pipeline

Stages and their progress values:

| Stage | Progress | Work |
|---|---|---|
| `analyse` | 0 | probe every asset; score sharpness and exposure; find faces for the focus point; drop near-duplicates; detect scene cuts (PySceneDetect `ContentDetector`) and propose candidate sub-clips in videos |
| `music` | 0.25 | choose or generate the track (library tracks are chosen from their metadata first; only the candidates are fetched), detect beats, downbeats and sections |
| `direct` | 0.35 | the director writes an EditPlan in beats; it is validated, repaired or replaced by the heuristic plan |
| `conform` | 0.45 | beats become frames: the Timeline |
| `render` | 0.5 | render each clip to a cached segment, join them, mix audio, burn text, encode |
| `finalise` | 0.97 | store the video, poster and Timeline; record usage |

Analysis and beat results are cached by file content under `MONTAJ_CACHE_DIR/analysis/` and `MONTAJ_CACHE_DIR/beats/`, so repeated jobs on the same media skip that work.

Video edits run the same stages. Renders start at `render`. Image edits run `direct` (prompt only), `render`, `finalise`.

### Directors

`make_director` returns the Claude director when an Anthropic key is configured, otherwise the heuristic director.

**Claude director** (`director/claude.py`):

- The system prompt (rules, style table, EditPlan contract) is sent with `cache_control` so repeated jobs reuse the prompt cache.
- One user turn carries the job (prompt, overrides, beat grid and section summary, the candidate list) and one labelled 512 px JPEG thumbnail per candidate, at most 120.
- The response is constrained to the EditPlan JSON Schema with structured outputs (`output_config` with `format: json_schema` and the configured `effort`). It is streamed, with a 10 minute timeout, 2 SDK retries and at most 32000 output tokens.
- Requests carry the beta flag `server-side-fallback-2026-07-01` with `fallbacks="default"`.
- A stop reason other than `end_turn`, or an invalid plan after two repair rounds, falls back to the heuristic plan (`source: "heuristic_fallback"`).

**Heuristic director** (`director/heuristic.py`): deterministic for a given seed. It orders candidates by score, drops near-duplicates, takes at most three shots from one video, sizes shots from the style and the music's energy, picks transitions and motion in rotation from the style, adds a short title from the prompt, and maps image prompts to operations by keywords.

Requests always win over the plan: `aspect`, `style` and the music asset in the request override what the director wrote, and `target_duration` refits the plan to that length.

### Rendering

1. **Normalise** each clip to a segment at the output canvas and frame rate: decode at `source_in` and `speed`, tone-map HDR, fit (`cover`, `contain`, `blur_fill`), apply motion by scaling a supersampled frame and cropping it, apply colour. Segments are intermediate H.264 files at a high quality.
2. **Cache** each segment under `MONTAJ_CACHE_DIR/<org_id>/seg_<key>.mp4`. The key is a SHA-256 over every field that changes the clip's pixels, the asset's content hash, the canvas, the frame rate, the quality and the renderer version. A revision that leaves a clip unchanged reuses it.
3. **Assemble** the segments with hard cuts or `xfade` transitions, in chunks of 8.
4. **Mix audio** at 48 kHz: clip sound, music ducked with `sidechaincompress`, fades, then two-pass `loudnorm` to -14 LUFS.
5. **Burn text** from an ASS file with libass.
6. **Encode** H.264 or H.265 at the quality's CRF and preset, AAC 192 kb/s, `+faststart`, and extract the poster frame.

Every FFmpeg and ffprobe call runs with a time limit of 60 s plus 10 s per second of media it handles (at most 4 h; 1 h for a call that gives none) and with `-protocol_whitelist file,pipe` on its inputs, so a crafted playlist or concat list cannot fetch a URL. The segment cache is shared: eviction never removes a file used in the last 10 minutes, and a segment is touched before it is validated and again before the final join.

Details are in [timeline.md](timeline.md#encoding).

## Webhooks and maintenance

Deliveries are rows in `webhook_deliveries`, written with the job's final state. Each worker process runs one delivery loop, separate from its job loops so a slow endpoint never delays job claims; it sends due deliveries (up to 10 per round, 4 in parallel), records the attempt and schedules the next one with exponential backoff. Event bodies carry no signed download URLs (receivers fetch the job for fresh ones). An org has at most `MONTAJ_MAX_WEBHOOKS_PER_ORG` active webhooks. Every 60 seconds a worker also deletes expired idempotency records and the stored bytes of assets deleted more than `MONTAJ_ASSET_PURGE_AFTER_HOURS` ago.

## Determinism

- The same input, assets and seed give the same EditPlan from the heuristic director and the same Timeline after conform. The seed is stored in the job input and output.
- The Claude director is not deterministic; its plan is returned in the Timeline, so a result can be reproduced by rendering that Timeline.
- The renderer is a fixed FFmpeg filter graph for a given Timeline. Output bytes can still differ between FFmpeg builds.
- The procedural bed is deterministic from style, length and seed.

## Testing and CI

`.github/workflows/ci.yml` runs on every push and pull request:

- **test** (Ubuntu, with a Postgres 17 service): installs FFmpeg, `uv sync --frozen --all-packages`, `ruff check`, `ruff format --check`, strict `mypy` on all four packages (`packages/core/src`, `apps/api/src`, `packages/sdk/src`, `apps/mcp/src`), `scripts/check_structure.py`, `pytest` (against SQLite), `montaj-api migrate` twice against Postgres, then `scripts/smoke_e2e.py --production --database-url ...` against the same Postgres.
- **windows**: FFmpeg from Chocolatey, then `pytest`.
- **docker**: builds the image, runs one SQLite container to healthy and ready, then starts the Compose stack with generated secrets and checks `/ready`, the migration exit code, the running worker and `admin create-key`.

`scripts/smoke_e2e.py` runs the whole system end to end with its own API and worker processes, generated media and the heuristic director: a montage checked with ffprobe, a revision that reuses segments, an image edit, a signed webhook delivery to a local receiver, MCP over stdio (`create_montage`, `get_job`) and `montaj montage --wait`. By default it uses a temporary SQLite database and local storage; `--database-url`, the `--s3-*` options and `--production` aim it at Postgres, S3 and production settings. See [deployment.md](deployment.md#verifying-a-deployment).

In the test suite S3 is covered by a fake client (`apps/api/src/montaj_api/common/__tests__/test_storage.py`). On the development machine the smoke script also ran in production mode against PostgreSQL (the `pgserver` package) and an S3-compatible server (moto). The Claude director is tested with a fake Anthropic client only.

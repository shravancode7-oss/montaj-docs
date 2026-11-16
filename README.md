# Montaj

![Montaj: your footage, cut on the beat](assets/poster-landscape-1920x1080.png)

> **This repository holds the documentation only.** The source code lives in a private repository, so the commands below need access to it. Paths such as `deploy/` and `scripts/` refer to that repository.

Montaj is a self-hostable service that edits images and videos. Its main feature turns your photos and short clips, plus a short prompt, into a finished MP4 whose cuts land on the beats of the music. You can drive it through a REST API, a Python SDK and `montaj` CLI, or an MCP server for coding assistants such as Claude Code.

## What makes it different

- **Beat-synced editing of your own media.** Montaj detects the beats, downbeats and sections of the track and cuts on them. It edits the photos and clips you upload; it does not generate footage.
- **A director plus a deterministic renderer.** The director plans the edit in musical beats, never in seconds, so it cannot ask for an off-beat cut. Claude Opus 5.5 (`claude-opus-5-5`) directs when an Anthropic API key is configured; otherwise a deterministic heuristic director follows the style presets. A deterministic FFmpeg renderer then executes the plan.
- **Editable timeline JSON with segment reuse.** Every video job returns its Timeline JSON and seed. Send back a patched Timeline and only the clips you changed are re-rendered; unchanged clips come from the segment cache.
- **Agent-native MCP surface.** Ten tools (upload, montage, edit, wait, download, revise and a few helpers) with structured results and signed URLs. Media bytes are never returned inline.
- **A licence record per render.** Every video output records where its music came from: a user upload, a library track with its licence and attribution, or the procedural bed.

## Local quickstart

You need [uv](https://docs.astral.sh/uv/) and FFmpeg (`ffmpeg` and `ffprobe` on `PATH`). Python 3.12 is installed by uv if missing. Run every command from the repository root: the API and the worker share `./data` (database, storage, cache) relative to the working directory.

```bash
uv sync --all-packages
uv run montaj-api migrate
uv run montaj-api admin create-key --name local     # prints the key once: mj_live_...
```

Start the API and the worker in two terminals:

```bash
uv run montaj-api serve      # http://127.0.0.1:8000 (OpenAPI UI at /docs)
uv run montaj-worker
```

In a third terminal, generate sample media and build a montage:

```bash
export MONTAJ_API_KEY=mj_live_...                    # PowerShell: $env:MONTAJ_API_KEY="mj_live_..."
uv run python scripts/gen_fixtures.py fixtures
uv run montaj montage fixtures/photo_1.jpg fixtures/photo_2.jpg fixtures/photo_3.jpg \
  fixtures/photo_4.jpg fixtures/photo_5.jpg fixtures/clip_1.mp4 fixtures/clip_2.mp4 \
  --prompt "upbeat summer trip" --music fixtures/music.wav --aspect 9:16 --quality draft \
  --out montage.mp4
```

The CLI uploads the files, queues the job, waits for it and downloads `montage.mp4`. Without an Anthropic key the heuristic director plans the edit. In development mode the server uses fixed, publicly known secrets and logs a warning; see [docs/deployment.md](docs/deployment.md) before exposing it. The full walkthrough, including curl and the SDK, is in [docs/quickstart.md](docs/quickstart.md).

To check a whole installation in one go, `uv run python scripts/smoke_e2e.py` starts its own API and worker in a temporary folder and runs a montage, a revision that reuses segments, an image edit, a signed webhook, MCP over stdio and `montaj montage --wait`; add `--production`, `--database-url` and the `--s3-*` options to aim it at Postgres and S3. See [Verifying a deployment](docs/deployment.md#verifying-a-deployment).

## Three ways to use it

**REST** (all endpoints under `/v1`, `Authorization: Bearer <key>`):

```bash
curl -s -X POST http://127.0.0.1:8000/v1/montages \
  -H "Authorization: Bearer $MONTAJ_API_KEY" -H "Content-Type: application/json" \
  -d '{"asset_ids": ["ast_...", "ast_..."], "prompt": "calm sunset walk", "aspect": "16:9"}'
```

**Python SDK and CLI** (`MONTAJ_API_URL`, default `http://127.0.0.1:8000`, and `MONTAJ_API_KEY`):

```python
from montaj import Montaj

client = Montaj()
assets = [client.assets.upload(p) for p in ["a.jpg", "b.jpg", "clip.mp4"]]
job = client.montages.create([a.id for a in assets], "a summer trip", aspect="9:16")
job = client.jobs.wait(job.id, timeout=900)
client.jobs.download_output(job, "montage.mp4")
```

**MCP** (stdio for a local assistant, or Streamable HTTP):

```bash
claude mcp add --transport stdio --env MONTAJ_API_KEY=mj_live_... montaj \
  -- uv run --directory /path/to/montaj montaj-mcp --root /path/to/media
```

Streamable HTTP requires a bearer token (`--auth-token` or `MONTAJ_MCP_TOKEN`); without one `montaj-mcp --transport http` refuses to start, unless `--allow-unauthenticated` is given on a loopback host. Register it with `claude mcp add --transport http montaj http://127.0.0.1:8765/mcp --header "Authorization: Bearer <token>"`. See [docs/mcp.md](docs/mcp.md).

## Features

- Montages from photos and/or clips plus a prompt, cut to detected beats, with optional music.
- Prompt-driven video edits: trim, join, reframe to 16:9, 9:16, 1:1 or 4:5, speed, colour, text and music.
- Explicit Timeline renders (`POST /v1/renders`) and a free estimate (`POST /v1/renders/estimate`).
- Revisions: a patched Timeline or a revision prompt becomes a new job that reuses cached segments.
- Image edits: resize, crop, face-aware smart crop, rotate, flip, auto enhance, adjust, denoise, colour preset, text, convert, and background removal with the optional `rembg` extra. Operations can be given directly or planned from a prompt.
- Seven style presets: cinematic, upbeat, calm, travel, celebration, product, minimal.
- Media analysis: sharpness, exposure clipping, faces for the focus point, near-duplicate rejection, scene cuts and candidate sub-clips in videos.
- An organisation music library with licence metadata; a procedural music bed as the fallback.
- Jobs with progress, cancellation, retries after a worker crash, and signed webhooks.
- API keys with `read`, `write` and `admin` scopes, idempotency keys, rate limits, a per-organisation job quota, usage metering, JSON logs and Prometheus metrics (optionally behind a bearer token).
- SQLite for local use, Postgres for production; local disk or S3 storage.
- A Docker image and a Compose stack.

## Documentation

- [Quickstart](docs/quickstart.md): local setup on Windows, macOS and Linux, first montage via CLI, curl and SDK.
- [REST API](docs/api.md): authentication, errors, idempotency, rate limits, every endpoint, jobs, webhooks.
- [Timeline](docs/timeline.md): the Timeline and EditPlan models, conform, transitions, rendering settings.
- [MCP server](docs/mcp.md): tools, Claude Code configuration, a worked example.
- [Styles and music](docs/styles-and-music.md): style presets, music selection, licence records.
- [Deployment](docs/deployment.md): Docker, Compose, every environment variable, operations, security.
- [Architecture](docs/architecture.md): components, data flow, job lifecycle, rendering stages, caching.
- [Licences](docs/licences.md): runtime dependencies and their licences, FFmpeg, optional extras.

## Current limits

- **Procedural music is a fallback.** When no track is supplied and the library has none, Montaj synthesises a simple bed (kick, hi-hat, bass, pad over a four-chord loop). It gives the edit a steady pulse; it is not a substitute for real music.
- **No music catalogue is bundled.** Add your own licensed tracks with `POST /v1/music` or `montaj music add`.
- **Image size is capped.** Image edits refuse sources over 144 megapixels and any operation that would grow the image past 12000 px per side or 144 megapixels. Uploads are held to `MONTAJ_MAX_MEDIA_DURATION_SECONDS` and `MONTAJ_MAX_MEDIA_DIMENSION`, and every FFmpeg call has a time limit scaled to the media (at most 4 h).
- **Rate limits are per API instance.** The token buckets live in process memory, so with N API instances the effective limit is N times the setting.
- **No generative features.** Generative video (text or image to video) and generative image edits (inpainting, restyling) are not included, and there is no generative provider interface in the code yet.
- **Not included either:** speech-to-text captions (text overlays exist, transcription does not), a web dashboard (the OpenAPI UI at `/docs` is the only UI), billing, SSO.
- **What was verified, and how.** On the development machine the full flow ran end to end with `scripts/smoke_e2e.py` in production mode against a real PostgreSQL server (started with the `pgserver` package) and an S3-compatible server (moto), and also on SQLite with local storage. The API test suite runs on SQLite; CI runs the migrations twice and the same smoke script in production mode against Postgres 17.
- **The Docker image and Compose stack were not built locally** (Docker is not installed on the development machine); they are built and exercised in CI.
- **The Claude director has been tested only with a fake client** (no Anthropic API key was available). The heuristic director is what ran end to end.

## Licence

The source code is private and not published here. Third-party components and their licences are listed in [docs/licences.md](docs/licences.md).

# Quickstart

This guide runs Montaj locally with SQLite and local file storage, creates an API key and renders a first montage three ways: with the `montaj` CLI, with curl, and with the Python SDK.

## 1. Requirements

| Requirement | Why |
|---|---|
| [uv](https://docs.astral.sh/uv/) | Installs Python 3.12 (`.python-version`) and the workspace. |
| FFmpeg with `ffprobe` on `PATH` | Every probe, decode and render is an `ffmpeg`/`ffprobe` subprocess. `/ready` reports `ffmpeg: error` without them. |

The renderer uses `libx264` (and `libx265` for `video_codec: "h265"`), the `subtitles` filter (libass) for text overlays, and `zscale` plus `tonemap` for HDR sources. Common FFmpeg builds include these.

Install FFmpeg:

- **Windows:** `winget install Gyan.FFmpeg` or `choco install ffmpeg` (CI uses Chocolatey). Open a new terminal afterwards so `PATH` is refreshed.
- **macOS:** `brew install ffmpeg`.
- **Linux (Debian, Ubuntu):** `sudo apt-get install ffmpeg`. The Docker image and CI install Debian's `ffmpeg` and `fonts-dejavu-core`.

Check with `ffmpeg -version` and `ffprobe -version`.

## 2. Install and migrate

From the repository root:

```bash
uv sync --all-packages
uv run montaj-api migrate
```

`migrate` applies the Alembic migrations (`alembic upgrade head`; `--revision` picks another target). With the default `MONTAJ_DATABASE_URL=sqlite+aiosqlite:///./data/montaj.db` it creates `./data/montaj.db`. Storage, work and cache folders default to `./data/storage`, `./data/work` and `./data/cache`.

All paths are relative to the working directory, so start the API, the worker and the admin commands from the same folder. The API server and the worker also read their settings from a `.env` file in that folder; copy the root `.env.template` to `.env` to start one. The template covers only the server and the worker: the SDK, the CLI and the MCP server never read `.env` (see step 6).

The workspace installs only `opencv-python-headless` 4.x: the root `pyproject.toml` excludes the GUI `opencv-python` wheel that scenedetect asks for, because OpenCV 5 has no Haar face cascades. If you synced this checkout before that change, run `uv sync --all-packages --reinstall-package opencv-python-headless` once; a test fails while face detection is unavailable.

## 3. Create the first API key

```bash
uv run montaj-api admin create-key --name local
```

Output: the key's JSON (`id`, `org_id`, `name`, `scopes`), then the secret on its own line:

```text
{
  "id": "key_...",
  "org_id": "org_...",
  "name": "local",
  "scopes": ["read", "write", "admin"]
}

API key (shown once, store it now):
mj_live_abcdefghijkl_...
```

Options: `--name` (default `admin`), `--scopes` (comma-separated, default `read,write,admin`), `--org-id` to add a key to an existing organisation, `--org-name` (default `default`) for a new one. Without `--org-id` a new organisation is created.

Only an HMAC of the secret is stored. If you lose it, create another key.

## 4. Start the API and the worker

Two terminals, both in the repository root:

```bash
uv run montaj-api serve
```

```bash
uv run montaj-worker
```

The API listens on `http://127.0.0.1:8000` (`MONTAJ_HOST`, `MONTAJ_PORT`, or `serve --host/--port`). The OpenAPI UI is at `http://127.0.0.1:8000/docs`. The worker claims jobs from the database; `--concurrency N` runs N job loops, `--once` runs a single tick and exits.

In development mode (`MONTAJ_ENV=development`, the default) the key pepper and the URL-signing secret fall back to fixed, publicly known values and a warning is logged. That is fine on your own machine; never expose such a server. See [deployment.md](deployment.md).

Check readiness:

```bash
curl -s http://127.0.0.1:8000/ready
# {"status":"ready","checks":{"database":"ok","storage":"ok","ffmpeg":"ok"}}
```

## 5. Generate sample media

```bash
uv run python scripts/gen_fixtures.py fixtures
```

This writes into `fixtures/`: `photo_1.jpg` to `photo_5.jpg` (1200x900, every third one portrait 900x1200), `clip_1.mp4` and `clip_2.mp4` (640x360, 30 fps, with a tone; `clip_2` has a hard scene cut half way) and `music.wav` (a 120 BPM beat with accented downbeats). It prints a JSON manifest of the paths. Options: `--photos 5`, `--clip-seconds 3`, `--music-seconds 20`.

## 6. First montage with the CLI

Set the key for the client (the CLI and SDK read `MONTAJ_API_KEY` and `MONTAJ_API_URL` from the environment, not from `.env`):

```bash
# bash/zsh
export MONTAJ_API_KEY=mj_live_...
# PowerShell
$env:MONTAJ_API_KEY = "mj_live_..."
```

Then:

```bash
uv run montaj montage fixtures/photo_1.jpg fixtures/photo_2.jpg fixtures/photo_3.jpg \
  fixtures/photo_4.jpg fixtures/photo_5.jpg fixtures/clip_1.mp4 fixtures/clip_2.mp4 \
  --prompt "upbeat summer trip" --music fixtures/music.wav --aspect 9:16 --quality draft \
  --out montage.mp4
```

Inputs may be local files (uploaded first) or existing `ast_...` asset ids. `--music` takes an audio file, an `ast_...` audio asset or a `mus_...` library track. Other options: `--style`, `--duration` (seconds), `--seed`, `--no-wait`, `--timeout` (default 1800 s), `--force` to overwrite the output, `--json` for machine-readable output.

`--quality draft` renders at half resolution (540x960 for 9:16) for a quick preview. Drop `--music` and Montaj picks a track from your organisation's library, or generates a procedural bed when the library is empty.

Exit codes: 0 ok, 1 API or connection error, 2 bad usage, 3 the job failed or was cancelled, 4 timed out waiting.

## 7. The same montage with curl

Upload each file (multipart field `file`):

```bash
curl -s -X POST http://127.0.0.1:8000/v1/assets \
  -H "Authorization: Bearer $MONTAJ_API_KEY" \
  -F "file=@fixtures/photo_1.jpg"
# {"id":"ast_...","object":"asset","kind":"image","mime_type":"image/jpeg",...}
```

Repeat for the other photos, the clips and `music.wav`. Then queue the montage:

```bash
curl -s -X POST http://127.0.0.1:8000/v1/montages \
  -H "Authorization: Bearer $MONTAJ_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: first-montage-1" \
  -d '{
        "asset_ids": ["ast_PHOTO1", "ast_PHOTO2", "ast_CLIP1"],
        "music_asset_id": "ast_MUSIC",
        "prompt": "upbeat summer trip",
        "aspect": "9:16",
        "quality": "draft"
      }'
# 202 {"id":"job_...","object":"job","type":"montage","status":"queued",...}
```

### Check the job

```bash
curl -s http://127.0.0.1:8000/v1/jobs/job_... -H "Authorization: Bearer $MONTAJ_API_KEY"
```

`status` goes `queued`, `running`, then `succeeded`, `failed` or `cancelled`. While running, `progress` (0 to 1) and `stage` (`analyse`, `music`, `direct`, `conform`, `render`, `finalise`) move forward. Poll every second or two, or register a webhook (see [api.md](api.md#webhooks)).

### Download the output

A succeeded job carries `output.video.download_url`, a short-lived signed URL (default 15 minutes) that needs no API key:

```bash
curl -s -o montage.mp4 "<output.video.download_url>"
```

Or follow the redirect from the authenticated content endpoint, which signs a fresh URL each time:

```bash
curl -s -L -o montage.mp4 http://127.0.0.1:8000/v1/assets/ast_VIDEO/content \
  -H "Authorization: Bearer $MONTAJ_API_KEY"
```

`output.poster` is a JPEG poster frame and `output.timeline` the rendered Timeline JSON.

## 8. The same montage with the Python SDK

```python
from pathlib import Path

from montaj import Montaj

client = Montaj()  # MONTAJ_API_KEY, MONTAJ_API_URL (default http://127.0.0.1:8000)

media = sorted(Path("fixtures").glob("photo_*.jpg")) + sorted(Path("fixtures").glob("clip_*.mp4"))
assets = [client.assets.upload(p) for p in media]
music = client.assets.upload("fixtures/music.wav")

job = client.montages.create(
    [a.id for a in assets],
    "upbeat summer trip",
    music_asset_id=music.id,
    aspect="9:16",
    quality="draft",
)
job = client.jobs.wait(
    job.id, timeout=900, on_update=lambda j: print(j.status, j.stage, j.progress)
)
print(job.director)  # {"source": "heuristic", "notes": [...], ...}
client.jobs.download_output(job, "montage.mp4")
client.jobs.download_output(job, "poster.jpg", poster=True)
```

`jobs.wait` raises `JobFailedError` when the job fails or is cancelled and `JobTimeoutError` when the timeout passes (the job keeps running). Every POST from the SDK carries an `Idempotency-Key`, reused across its automatic retries, so a retried request never creates a second job.

## 9. Enable the Claude director

Set an Anthropic API key where the **worker** can read it (the worker runs the director):

```bash
export ANTHROPIC_API_KEY=sk-ant-...      # or MONTAJ_ANTHROPIC_API_KEY
uv run montaj-worker
```

You can also put `ANTHROPIC_API_KEY=...` in `.env` in the working directory. Optional settings: `MONTAJ_DIRECTOR_MODEL` (default `claude-opus-5-5`) and `MONTAJ_DIRECTOR_EFFORT` (default `high`; accepted values `low`, `medium`, `high`, `xhigh`, `max`).

With a key, Claude plans the edit from the prompt, the style table, a beat and section summary, and one labelled 512 px thumbnail per candidate shot (at most 120). An invalid plan gets up to two repair rounds; after that, or on a refusal or API error, the heuristic plan is used. The job output tells you which one ran:

```json
"director": {"source": "claude", "notes": ["..."], "input_tokens": 12345, "output_tokens": 678}
```

`source` is `claude`, `heuristic` (no key), `heuristic_fallback` (Claude failed or its plan was invalid) or `none` (explicit renders and image edits with operations). Token counts are metered as usage (`director_input_tokens`, `director_output_tokens`).

## Next steps

- Iterate on a result: `uv run montaj revise job_... --prompt "slower cuts, no title"`.
- Add licensed music: `uv run montaj music add track.mp3 --title "Sunrise" --licence CC-BY-4.0 --tags upbeat,travel` (optionally `--tempo 118 --energy 0.7`; the tempo is measured from the audio when omitted).
- Drive it from Claude Code: [mcp.md](mcp.md).
- Run the end-to-end smoke test: `uv run python scripts/smoke_e2e.py` (boots its own API and worker in a temporary folder, uploads generated media and, with the heuristic director, runs a 9:16 draft montage checked with ffprobe, a revision that reuses segments, an image edit, a signed webhook, MCP over stdio and `montaj montage --wait`). Options and what has been verified: [deployment.md](deployment.md#verifying-a-deployment).

# MCP server

`montaj-mcp` exposes Montaj to MCP clients such as Claude Code. It is a thin layer over the Python SDK: every tool calls the REST API, so the API and a worker must be running (see [quickstart.md](quickstart.md)).

Tools return small structured results with ids and signed URLs. Media bytes are never returned inline; `download_output` writes files to disk instead.

## Running it

```text
montaj-mcp [--transport stdio|http] [--host 127.0.0.1] [--port 8765] [--path /mcp]
           [--auth-token TOKEN] [--allow-unauthenticated]
           [--root ROOT] [--log-level DEBUG|INFO|WARNING|ERROR|CRITICAL] [--version]
```

| Option | Default | Meaning |
|---|---|---|
| `--transport` | `stdio` | `stdio` for a local client that starts the process; `http` serves Streamable HTTP |
| `--host`, `--port`, `--path` | `127.0.0.1`, `8765`, `/mcp` | HTTP only: the endpoint is `http://127.0.0.1:8765/mcp` |
| `--auth-token` | none | HTTP only: the bearer token every client must send. Prefer `MONTAJ_MCP_TOKEN`: a command-line value is visible to other local users |
| `--allow-unauthenticated` | off | HTTP only: serve without a token. Accepted only with a loopback `--host` (`localhost` or a loopback address) |
| `--root` | stdio: none; http: the current working directory | confine local reads (`upload_asset`) and writes (`download_output`) to this folder; relative paths resolve against it |
| `--log-level` | `WARNING` | logs go to stderr |

Environment: `MONTAJ_API_URL` (default `http://127.0.0.1:8000`) and `MONTAJ_API_KEY`, read from this process's environment only (`.env` files are not read), and `MONTAJ_MCP_TOKEN` for the HTTP bearer token. The SDK client is created on the first tool call, so a missing key shows up as a tool error, not a start-up failure. The key needs the `write` scope for the tools that create things.

With `--transport http`, `--root` defaults to the server's current working directory, so local file access over HTTP is always confined. With stdio and no `--root`, relative paths resolve against the working directory and any path is allowed; set `--root` whenever the client is not fully trusted. Paths are taken literally: environment variables such as `$HOME` or `%USERPROFILE%` are not expanded (a leading `~` still is). Error messages echo only the path as given, never the resolved path or the root.

The HTTP transport requires `Authorization: Bearer <token>` on every request, with the token from `--auth-token` or `MONTAJ_MCP_TOKEN`; a request without it, or with another token, gets 401 with `WWW-Authenticate: Bearer` and the REST API's error envelope (`{"error": {"code": "unauthorized", "message", "details": null, "request_id": null}}`). Without a token `--transport http` refuses to start (exit status 2), unless `--allow-unauthenticated` is given and `--host` is a loopback address; then it starts with a warning, and any local process can act with the configured API key. Whoever holds the token acts with that API key, so give the server a key with only the scopes it needs, keep it on `127.0.0.1` or behind TLS, and set `--root` to the folder it should use.

## Claude Code configuration

### stdio (recommended for local use)

```bash
claude mcp add --transport stdio \
  --env MONTAJ_API_URL=http://127.0.0.1:8000 \
  --env MONTAJ_API_KEY=mj_live_... \
  montaj -- uv run --directory /path/to/montaj montaj-mcp --root /path/to/media
```

On Windows, use a Windows path such as `D:\path\to\montaj`. The equivalent JSON (for `.mcp.json` or another client's config):

```json
{
  "mcpServers": {
    "montaj": {
      "command": "uv",
      "args": ["run", "--directory", "/path/to/montaj", "montaj-mcp", "--root", "/path/to/media"],
      "env": {"MONTAJ_API_URL": "http://127.0.0.1:8000", "MONTAJ_API_KEY": "mj_live_..."}
    }
  }
}
```

### Streamable HTTP

Pick a token and start the server yourself:

```bash
export MONTAJ_MCP_TOKEN="$(python -c "import secrets; print(secrets.token_hex(32))")"
MONTAJ_API_KEY=mj_live_... uv run montaj-mcp --transport http --root /path/to/media
```

Then register it with the same token:

```bash
claude mcp add --transport http montaj http://127.0.0.1:8765/mcp \
  --header "Authorization: Bearer <token>"
```

The equivalent JSON entry is `{"type": "http", "url": "http://127.0.0.1:8765/mcp", "headers": {"Authorization": "Bearer <token>"}}`.

## Tools

The server sends these instructions to the client: upload each file (or reuse ids from `list_assets`), call `create_montage`, `edit_video` or `edit_image`, which return a `job_id` at once, call `get_job(job_id, wait_seconds=60)` until `done` is true, then `download_output`. Use `quality="draft"` for quick previews and `revise` to iterate.

| Tool | Arguments (default) | Returns |
|---|---|---|
| `upload_asset` | `path_or_url`: a local path or a public `http(s)` URL (imported by the server); `filename` (the file's name) | asset |
| `list_assets` | `kind` (`image`, `video`, `audio`); `limit` 1 to 200 (20); `cursor` | assets plus `next_cursor` |
| `create_montage` | `asset_ids` (images and/or clips); `prompt`; `music` (`ast_...` audio asset or `mus_...` library track; omit to pick one); `aspect` (omit: the aspect the prompt names, else the style's, and the heuristic director switches a landscape style to 9:16 when at least two thirds of the media is portrait); `duration` 3 to 600 s; `style`; `quality` (`standard`); `seed` (repeatable result with the heuristic director) | job |
| `edit_video` | the same as `create_montage`, plus `keep_audio` (true). `music` is added only when given; `aspect` omitted follows the prompt, else the first video's shape | job |
| `edit_image` | `asset_id`; exactly one of `prompt` or `operations` (see [api.md](api.md#image-edits)) | job |
| `get_job` | `job_id`; `wait_seconds` (0; capped at 60) to block until done; `include_timeline` (false) | job |
| `revise` | `job_id` of a succeeded job; exactly one of `prompt` or `timeline` | job |
| `download_output` | `job_id`; `dest_path` (a file path or an existing folder); `overwrite` (false); `poster` (false: save the poster JPEG instead) | `{job_id, path, size_bytes, asset_id, mime_type}` |
| `list_music` | `tag`; `limit` 1 to 500 (50) | `{tracks: [{track_id, asset_id, title, licence, attribution, tags, tempo, energy, duration}]}` |
| `get_timeline_schema` | none | the Timeline JSON Schema |

An asset result has `asset_id`, `kind`, `filename`, `mime_type`, `width`, `height`, `duration`, `has_audio`, `size_bytes` and `download_url`.

A job result:

| Field | Meaning |
|---|---|
| `job_id`, `type`, `status` | `status` is `queued`, `running`, `succeeded`, `failed` or `cancelled` |
| `done` | true once the job has finished, whatever the outcome |
| `progress`, `stage` | 0 to 1, and the current stage |
| `error` | code, message and details when the job failed |
| `output_asset_id`, `video_url`, `image_url`, `poster_url` | the output and short-lived signed URLs |
| `duration`, `seed` | output length and the seed to reproduce it |
| `director_source`, `director_notes` | who planned the edit (`claude`, `heuristic`, `heuristic_fallback`) and its notes |
| `music` | a one-line summary of the music's licence record |
| `segments_reused` | clips taken from the render cache |
| `timeline` | the rendered Timeline, only with `include_timeline=true` |
| `next_step` | a sentence telling the agent what to call next |

A job that fails is a normal result (`status: "failed"` with `error`), not a tool error. Tool errors are raised for bad arguments, API errors (with the API's code and message), unreadable files and paths outside `--root`.

## Example session

A user asks Claude Code: "Make a 20 second vertical reel from the photos in ./trip with the track in ./trip/song.mp3."

1. `upload_asset(path_or_url="trip/beach.jpg")` and so on for each photo and the song. Each returns an `asset_id`.
2. `create_montage(asset_ids=["ast_A", "ast_B", "ast_C", "ast_D"], prompt="summer trip reel", music="ast_SONG", aspect="9:16", duration=20, quality="draft")` returns `{"job_id": "job_...", "status": "queued", "done": false, "next_step": "Not finished. Call get_job(...)"}`.
3. `get_job(job_id="job_...", wait_seconds=60)` until `done` is true. On success it returns `video_url`, `duration`, `seed`, `director_source` and the music licence summary.
4. `download_output(job_id="job_...", dest_path="trip/reel_draft.mp4")`.
5. The user says "slower and no title". `revise(job_id="job_...", prompt="slower cuts, no title")` returns a new job; repeat steps 3 and 4. Clips whose pixels did not change are reused (`segments_reused`).
6. For the final version, call `create_montage` again with `quality="standard"` and the same `seed`, or `get_job(..., include_timeline=true)`, patch `timeline.output.quality`, and `revise(job_id=..., timeline=...)`. A patched Timeline keeps its own canvas size, so set `output.width` and `output.height` to full size as well.

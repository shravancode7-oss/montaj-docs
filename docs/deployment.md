# Deployment

Montaj runs as two processes from one image or one checkout:

- **API** (`montaj-api serve`): HTTP, validation, storage of uploads, enqueuing jobs.
- **Worker** (`montaj-worker`): runs jobs, delivers webhooks (in its own loop), purges expired data.

They share the database (which is also the job queue) and the storage. Run as many of each as you need; every API instance and every worker must use the same `MONTAJ_DATABASE_URL` and the same storage settings and secrets.

The Docker image and Compose stack described here are built and exercised in CI (`.github/workflows/ci.yml`). They were not built on the development machine, where Docker is not installed. The application itself was run there end to end in production mode against PostgreSQL and an S3-compatible server; see [Verifying a deployment](#verifying-a-deployment).

## Before production: required settings

With `MONTAJ_ENV=production`:

- `MONTAJ_KEY_PEPPER` and `MONTAJ_URL_SIGNING_SECRET` are required and must each be at least 32 characters. The publicly known development values are refused even when set explicitly. Start-up fails otherwise. Generate them with:

  ```bash
  python -c "import secrets; print(secrets.token_hex(32))"
  ```

- The pepper is part of every stored key hash. Changing it invalidates every API key. Changing the signing secret invalidates outstanding signed download URLs.
- `MONTAJ_STORAGE_BACKEND=s3` requires `MONTAJ_S3_BUCKET`.
- Set `MONTAJ_PUBLIC_BASE_URL` to the address clients use; local-storage download links are built from it.
- An unset `MONTAJ_METRICS_TOKEN` (with metrics enabled) logs a warning at start-up: `/metrics` and the worker metrics port are then open to anyone who can reach them.

In development (the default) both secrets fall back to fixed, publicly known values and a warning is logged. Never expose a development-mode server.

## Docker image

```bash
docker build -f deploy/Dockerfile -t montaj:latest .
```

`deploy/Dockerfile`:

- Build stage: `python:3.12-slim-bookworm` with uv 0.11.9, `uv sync --frozen --no-dev --all-packages`.
- Runtime: `python:3.12-slim-bookworm` plus Debian's `ffmpeg`, `fonts-dejavu-core`, `fontconfig` and `tini`.
- Runs as the non-root user `montaj` (uid and gid 10001). `VOLUME /data`, `EXPOSE 8000`.
- Defaults: `MONTAJ_ENV=production`, `MONTAJ_HOST=0.0.0.0`, `MONTAJ_PORT=8000`, `MONTAJ_DATABASE_URL=sqlite+aiosqlite:////data/montaj.db`, `MONTAJ_STORAGE_DIR=/data/storage`, `MONTAJ_WORK_DIR=/data/work`, `MONTAJ_CACHE_DIR=/data/cache`, `NUMBA_CACHE_DIR=/tmp/numba-cache`.
- `HEALTHCHECK` calls `/health` every 30 s. `ENTRYPOINT ["/usr/bin/tini", "--"]`, `CMD ["montaj-api", "serve"]`.

The same image runs every command: `montaj-api migrate`, `montaj-api admin create-key`, `montaj-worker`.

### Single container with SQLite

Good for one machine. The API does **not** migrate on start, so run the migration first:

Put the two secrets in an env file once and keep it; every container must use the same values:

```bash
docker volume create montaj-data
printf 'MONTAJ_KEY_PEPPER=%s\nMONTAJ_URL_SIGNING_SECRET=%s\nMONTAJ_PUBLIC_BASE_URL=http://localhost:8000\n' \
  "$(openssl rand -hex 32)" "$(openssl rand -hex 32)" > montaj.env
docker run --rm --env-file montaj.env -v montaj-data:/data montaj:latest montaj-api migrate
docker run -d --name montaj-api --env-file montaj.env -v montaj-data:/data -p 8000:8000 montaj:latest
docker run -d --name montaj-worker --env-file montaj.env -v montaj-data:/data montaj:latest montaj-worker
docker exec montaj-api montaj-api admin create-key --name first
```

SQLite runs in WAL mode, so one API and one worker on the same volume are fine. For several machines, use Postgres.

## Docker Compose (Postgres)

`deploy/docker-compose.yml` (project name `montaj`) runs:

| Service | What it does |
|---|---|
| `postgres` | `postgres:17`, health-checked with `pg_isready` (every 5 s, 20 retries), data in the `postgres-data` volume |
| `migrate` | one-shot `montaj-api migrate`, after Postgres is healthy |
| `api` | `montaj-api serve` on port 8000, health-checked on `/health` every 15 s; starts after `migrate` completed successfully |
| `worker` | `montaj-worker`; `stop_grace_period: 5m` so running jobs can finish; its image health check is disabled |

`api`, `worker` and `migrate` share the `montaj-data` volume at `/data` and read `deploy/.env`. `MONTAJ_DATABASE_URL` defaults to `postgresql+asyncpg://${POSTGRES_USER:-montaj}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB:-montaj}`.

```bash
cp deploy/.env.template deploy/.env
# edit deploy/.env: set POSTGRES_PASSWORD, MONTAJ_KEY_PEPPER, MONTAJ_URL_SIGNING_SECRET
docker compose -f deploy/docker-compose.yml up -d --build
docker compose -f deploy/docker-compose.yml exec api montaj-api admin create-key --name first
curl -s http://localhost:8000/ready
```

More workers:

```bash
docker compose -f deploy/docker-compose.yml up -d --scale worker=3
```

To use S3 instead of the shared volume, set `MONTAJ_STORAGE_BACKEND=s3` and the `MONTAJ_S3_*` values in `deploy/.env`. The work and cache folders stay on local disk in each container.

## Database

- **SQLite** (`sqlite+aiosqlite:///path`): the default. WAL journal. Jobs are claimed with an atomic `UPDATE ... WHERE status = 'queued' ... RETURNING`.
- **Postgres** (`postgresql+asyncpg://...`): for several API or worker hosts. Jobs are claimed with `SELECT ... FOR UPDATE SKIP LOCKED`.

Migrations are Alembic (`apps/api/migrations`, currently `0001` and `0002_idempotency_per_key`; the latter scopes idempotency records to the API key and drops existing records, which are only a 24 h replay cache): run `montaj-api migrate` (or `montaj-api admin migrate`) before starting a new version. Running it twice is safe; CI checks that.

## Storage

- **local** (default): files under `MONTAJ_STORAGE_DIR`. Download links point at `<MONTAJ_PUBLIC_BASE_URL>/v1/assets/{id}/file` with an HMAC signature, so the API serves the bytes. All API and worker processes must see the same folder.
- **s3**: objects in `MONTAJ_S3_BUCKET` under `MONTAJ_S3_PREFIX`. Download links are S3 presigned GETs. `MONTAJ_S3_ENDPOINT_URL` points at an S3-compatible service. Without `MONTAJ_S3_ACCESS_KEY_ID` and `MONTAJ_S3_SECRET_ACCESS_KEY`, boto3 uses its usual credential chain (environment, profile, instance role).

The S3 backend has a unit test with a fake S3 client, and the end-to-end smoke script ran against an S3-compatible server (moto) on the development machine. It has not been run against AWS S3 itself.

Disk use besides storage:

- `MONTAJ_WORK_DIR`: per-job scratch folders, removed after the job.
- `MONTAJ_CACHE_DIR`: media analysis, beat grids and rendered clip segments. After every job the worker trims the whole cache to `MONTAJ_CACHE_MAX_BYTES` (default 20 GiB), least recently used first. `0` means no cap.

## Environment variables

These are the settings of the API server and the worker. Every setting is `MONTAJ_<NAME>` and is also read from a `.env` file in the working directory (the root `.env.template` is a local-development starting point for it; `deploy/.env.template` is the Compose file and lists every setting with its default). Unknown variables are ignored. List values are JSON (`["https://app.example.com"]`). An empty value is not the same as unset: `MONTAJ_S3_BUCKET=` sets an empty string.

### Runtime and HTTP

| Variable | Default | Notes |
|---|---|---|
| `MONTAJ_ENV` | `development` | `development` or `production` |
| `MONTAJ_LOG_LEVEL` | `INFO` | |
| `MONTAJ_HOST` | `127.0.0.1` | bind address of the API and of the worker metrics server (`0.0.0.0` in the image and in `deploy/.env.template`) |
| `MONTAJ_PORT` | `8000` | |
| `MONTAJ_PUBLIC_BASE_URL` | `http://127.0.0.1:8000` | external address, used in local download links |
| `MONTAJ_FORWARDED_ALLOW_IPS` | `127.0.0.1` | passed to uvicorn as `forwarded_allow_ips`: the proxies (comma-separated IPs or CIDRs, `*` for any) whose `X-Forwarded-For` and `X-Forwarded-Proto` are trusted. The client IP drives the unauthenticated rate limit, so list only your proxies |
| `MONTAJ_CORS_ORIGINS` | `[]` | JSON list; empty disables CORS |
| `MONTAJ_METRICS_ENABLED` | `true` | serve `/metrics` |
| `MONTAJ_METRICS_TOKEN` | none | when set, `/metrics` on the API and the worker metrics port require `Authorization: Bearer <token>` (401 with the standard error envelope and `WWW-Authenticate: Bearer` otherwise); unset or empty, they are open, for private networks only, and production logs a warning |

### Database

| Variable | Default |
|---|---|
| `MONTAJ_DATABASE_URL` | `sqlite+aiosqlite:///./data/montaj.db` |
| `MONTAJ_DATABASE_ECHO` | `false` (log SQL) |

### Secrets and keys

| Variable | Default | Notes |
|---|---|---|
| `MONTAJ_KEY_PEPPER` | none | required in production, at least 32 characters, not the development value |
| `MONTAJ_URL_SIGNING_SECRET` | none | required in production, at least 32 characters, not the development value |
| `MONTAJ_KEY_ENV` | `live` | the `<env>` part of new keys, 1 to 8 lowercase letters or digits; keys with another env tag are rejected |
| `MONTAJ_SIGNED_URL_TTL_SECONDS` | `900` | 10 to 604800 |

### Storage

| Variable | Default |
|---|---|
| `MONTAJ_STORAGE_BACKEND` | `local` (`local` or `s3`) |
| `MONTAJ_STORAGE_DIR` | `./data/storage` |
| `MONTAJ_S3_BUCKET` | none (required for `s3`) |
| `MONTAJ_S3_PREFIX` | empty |
| `MONTAJ_S3_REGION` | none |
| `MONTAJ_S3_ENDPOINT_URL` | none |
| `MONTAJ_S3_ACCESS_KEY_ID`, `MONTAJ_S3_SECRET_ACCESS_KEY` | none (boto3 credential chain) |
| `MONTAJ_WORK_DIR` | `./data/work` |
| `MONTAJ_CACHE_DIR` | `./data/cache` |
| `MONTAJ_CACHE_MAX_BYTES` | `21474836480` (20 GiB; 0 = no cap) |

### Uploads and media

| Variable | Default |
|---|---|
| `MONTAJ_MAX_UPLOAD_BYTES` | `536870912` (512 MiB) |
| `MONTAJ_MAX_JSON_BODY_BYTES` | `2097152` (2 MiB) |
| `MONTAJ_MAX_MEDIA_DURATION_SECONDS` | `900` |
| `MONTAJ_MAX_MEDIA_DIMENSION` | `8192` |
| `MONTAJ_IMPORT_TIMEOUT_SECONDS` | `60` |
| `MONTAJ_IMPORT_MAX_REDIRECTS` | `3` (0 to 10) |
| `MONTAJ_ALLOW_PRIVATE_URLS` | `false`; `true` disables the private-address check for URL imports and webhooks (tests and local development only) |
| `MONTAJ_ASSET_PURGE_AFTER_HOURS` | `24`: bytes of deleted assets are removed after this |

### Rate limits and quotas

| Variable | Default |
|---|---|
| `MONTAJ_RATE_LIMIT_PER_MINUTE` | `120` (per key, per API instance) |
| `MONTAJ_RATE_LIMIT_BURST` | `60` |
| `MONTAJ_MAX_CONCURRENT_JOBS_PER_ORG` | `4` (queued plus running) |
| `MONTAJ_IDEMPOTENCY_TTL_HOURS` | `24` |

### Jobs and worker

| Variable | Default |
|---|---|
| `MONTAJ_JOB_MAX_ATTEMPTS` | `3` |
| `MONTAJ_JOB_HEARTBEAT_SECONDS` | `5` |
| `MONTAJ_JOB_STALE_AFTER_SECONDS` | `60` |
| `MONTAJ_WORKER_POLL_SECONDS` | `1` |
| `MONTAJ_WORKER_CONCURRENCY` | `1` (1 to 64; `--concurrency` overrides) |
| `MONTAJ_WORKER_METRICS_PORT` | none; set (for example `9100`) to serve worker metrics on `MONTAJ_HOST` |

### Webhooks

| Variable | Default |
|---|---|
| `MONTAJ_MAX_WEBHOOKS_PER_ORG` | `10` (at least 1): active webhooks per organisation; one more is 409 `webhook_limit_exceeded` |
| `MONTAJ_WEBHOOK_TIMEOUT_SECONDS` | `10` |
| `MONTAJ_WEBHOOK_MAX_ATTEMPTS` | `8` |
| `MONTAJ_WEBHOOK_BACKOFF_BASE_SECONDS` | `30` |
| `MONTAJ_WEBHOOK_BACKOFF_MAX_SECONDS` | `3600` |

### Director

| Variable | Default |
|---|---|
| `MONTAJ_ANTHROPIC_API_KEY` or `ANTHROPIC_API_KEY` | none: the heuristic director is used |
| `MONTAJ_DIRECTOR_MODEL` | `claude-opus-5-5` |
| `MONTAJ_DIRECTOR_EFFORT` | `high` (`low`, `medium`, `high`, `xhigh`, `max`) |

The worker calls the director, so the key must be set on the workers. The API does not need it.

### Client side

The SDK, the `montaj` CLI and `montaj-mcp` read `MONTAJ_API_URL` (default `http://127.0.0.1:8000`) and `MONTAJ_API_KEY` from their own process environment only. They do not read `.env`, and neither `.env.template` covers them: export them in the shell (or the MCP client's `env` block) that starts the client. `montaj-mcp --transport http` also reads `MONTAJ_MCP_TOKEN`; see [mcp.md](mcp.md#running-it).

## The worker

```text
montaj-worker [--concurrency N] [--once]
```

Each job loop iteration:

1. Re-queues jobs whose heartbeat is older than `MONTAJ_JOB_STALE_AFTER_SECONDS` (or fails them with `heartbeat_lost` after the last attempt).
2. Every 60 seconds: deletes expired idempotency records and purges the bytes of assets deleted more than `MONTAJ_ASSET_PURGE_AFTER_HOURS` ago.
3. Claims and runs one job, then sleeps `MONTAJ_WORKER_POLL_SECONDS` when there was nothing to do.

Next to the job loops, every worker process runs one webhook delivery loop: it sends due deliveries (up to 10 per round, 4 at a time) and sleeps `MONTAJ_WORKER_POLL_SECONDS` when none were due. Because it is separate, a slow webhook endpoint never delays job claims. Webhooks are only delivered while at least one worker runs. `--once` runs one job tick and one delivery round.

Shutdown: the first SIGINT or SIGTERM stops claiming jobs and starting delivery rounds and lets running ones finish; a second one exits at once. A job interrupted by a hard stop is re-queued once its heartbeat goes stale; an interrupted delivery is retried after its lease expires. Give the worker a long stop grace period (Compose uses 5 minutes).

Each loop runs one job at a time; `--concurrency N` runs N loops in one process. FFmpeg itself uses several cores per render, so raise concurrency with care on small machines.

## Observability

- **Health:** `GET /health` (process alive) and `GET /ready` (database, storage write, ffmpeg and ffprobe; 503 when any fails; the result is cached for 5 s). Use `/ready` for load-balancer checks.
- **Logs:** one JSON object per line on stdout with `ts`, `level`, `logger`, `message`, plus `request_id` and `job_id` when set. Fields whose name contains `secret`, `token`, `authorization`, `password`, `pepper`, `api_key` or `signature` are redacted. Uvicorn's access log is off; requests are logged by the request middleware.
- **Metrics:** `GET /metrics` on the API (unless disabled) and on `MONTAJ_WORKER_METRICS_PORT` for workers.

| Metric | Type |
|---|---|
| `montaj_http_requests_total` | counter |
| `montaj_http_request_duration_seconds` | histogram |
| `montaj_rate_limited_total` | counter |
| `montaj_jobs_enqueued_total` | counter |
| `montaj_jobs_finished_total` | counter |
| `montaj_job_duration_seconds` | histogram |
| `montaj_jobs` | gauge of jobs by status, read from the database at scrape time |
| `montaj_webhook_deliveries_total` | counter |

`/metrics` takes no API key. Set `MONTAJ_METRICS_TOKEN` on the API and the workers and configure the scraper to send `Authorization: Bearer <token>` (Prometheus: `authorization: {credentials: <token>}` in the scrape config). A wrong or missing token gets 401 `unauthorized` in the API's standard error envelope with `WWW-Authenticate: Bearer`. Without the token both endpoints are open (production logs a warning at start-up), so either set it or block them at your proxy. The worker metrics server binds `MONTAJ_HOST`: `127.0.0.1` by default on bare metal, `0.0.0.0` under Compose (from `deploy/.env.template`) so Prometheus can reach it on the container network.

## Security notes

- Only HMAC-SHA256 hashes of API keys are stored. Secrets are shown once.
- Uploads are identified by content, probed with ffprobe, checked against the duration and resolution limits from the probe (before anything is decoded) and then decoded once before they are accepted. Job outputs have a hard ceiling of 16384 px per side and 6 h.
- URL imports and webhook targets pass an SSRF guard: only public addresses, the connection pinned to the vetted address, every redirect re-checked (webhooks never follow redirects).
- FFmpeg runs as a subprocess with an argument list, never through a shell. Every FFmpeg and ffprobe call has a time limit (60 s plus 10 s per media second, at most 4 h; 1 h when a call gives none) and its inputs are restricted with `-protocol_whitelist file,pipe`, so a crafted playlist or concat list cannot make it fetch a URL.
- Run behind a TLS-terminating proxy. Uvicorn reads `X-Forwarded-For` and `X-Forwarded-Proto` (`proxy_headers=True`) only from the addresses in `MONTAJ_FORWARDED_ALLOW_IPS` (default `127.0.0.1`). Set it to your proxy's address (for example its Docker network address), never `*` when clients can reach the API directly, and only let the proxy reach the API port.
- Requests without a valid key, failed authentications included, take from the client IP's bucket; once it is empty, keys are no longer looked up for that IP, so an invalid-key flood is answered 429 without touching the database.
- Rate limits live in each API process. Put a shared limit in your proxy if you run several instances.
- Set `MONTAJ_METRICS_TOKEN`, or keep `/metrics` and the worker metrics port on a private network.
- `montaj-mcp --transport http` requires a bearer token (`MONTAJ_MCP_TOKEN` or `--auth-token`) and refuses to start without one, unless `--allow-unauthenticated` is given with a loopback host. Whoever holds the token acts with the server's API key, so give it a key with only the scopes it needs and set `--root`. See [mcp.md](mcp.md).

## Verifying a deployment

`scripts/smoke_e2e.py` checks a whole installation with real processes, real HTTP and real FFmpeg:

```text
uv run python scripts/smoke_e2e.py [--keep] [--timeout 600] [--production]
    [--database-url URL]
    [--s3-bucket B] [--s3-endpoint-url URL] [--s3-access-key K] [--s3-secret-key S] [--s3-region R]
```

| Option | Meaning |
|---|---|
| (none) | SQLite and local storage in a temporary folder |
| `--database-url` | use this database for the API and worker; it is migrated first and a new organisation is created, so a shared database is fine |
| `--s3-bucket` | use S3 storage in this bucket, under a fresh `smoke-<random>/` prefix; `--s3-endpoint-url` (MinIO and other S3-compatible services), `--s3-access-key`, `--s3-secret-key` and `--s3-region` complete it |
| `--production` | `MONTAJ_ENV=production` with freshly generated 32-byte secrets |
| `--timeout` | per-job timeout in seconds (default 600) |
| `--keep` | keep the temporary workspace |

Every child process runs in the temporary folder (so a developer `.env` is never read) with the Anthropic key blanked, so the heuristic director plans every edit. The script:

1. runs `montaj-api migrate` and `montaj-api admin create-key`;
2. starts `montaj-api serve` and `montaj-worker` and waits for `/health`;
3. registers a webhook pointing at a local receiver;
4. uploads 5 photos and 2 clips generated by `scripts/gen_fixtures.py`;
5. requests a 9:16 draft montage over REST, downloads it and checks it with ffprobe (540x960, an audio stream, longer than 5 s);
6. submits a revision with one clip's colour changed, which re-renders that clip and reuses the other segments;
7. runs an image edit with explicit operations;
8. checks that the receiver got a correctly signed `job.succeeded` webhook for the montage;
9. runs `montaj-mcp` over stdio (`create_montage`, then `get_job` until done);
10. runs `montaj montage --wait` (the CLI), which builds and downloads a montage.

It prints `SMOKE OK` and exits 0; on any failure it prints the reason and the tails of the process logs and exits 1. The subprocesses are always stopped.

What has been verified so far:

- On the development machine the full script ran in production mode against a real PostgreSQL server (started with the `pgserver` package) and an S3-compatible server (moto), and also on SQLite with local storage.
- CI (`.github/workflows/ci.yml`, job **test**) runs ruff, mypy on all four packages, the structure check and pytest on SQLite, then applies the migrations twice against a Postgres 17 service and runs the smoke script with `--production --database-url` against it. The **docker** job builds the image, brings one SQLite container to healthy and ready, and starts the Compose stack (Postgres, migrate, api, worker) with generated secrets.
- The Docker image and the Compose stack were not built on the development machine (no Docker there); only CI builds them.
- The Claude director has been tested only with a fake Anthropic client, since no API key was available. The heuristic director is what ran end to end.

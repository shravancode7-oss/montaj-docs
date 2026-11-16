# Security policy

## Reporting a vulnerability

Please do not open a public issue for a security problem. Use GitHub's private vulnerability reporting on this repository (Security tab, "Report a vulnerability"). Include the affected version or commit, steps to reproduce, and the impact you expect. You should get a first reply within five working days.

## Supported versions

Only the latest commit on `main` receives security fixes.

## What the project does to stay safe

- API keys are stored as HMAC-SHA256 hashes with a server-side pepper and compared in constant time.
- Every query is scoped to the caller's organisation; scopes (read, write, admin) are checked on every route.
- URL imports and webhooks go through an SSRF guard that blocks private and internal addresses and re-checks every redirect.
- FFmpeg is run with argument lists (never a shell), a timeout on every call, and inputs restricted to local file protocols.
- Uploads and image edits have size, dimension and duration limits that are checked before decoding.
- Production mode refuses to start with missing, short or publicly known development secrets.
- The MCP server over HTTP requires a bearer token and is confined to a root folder.
- CI runs lint, type checks, the test suite, an end-to-end run on PostgreSQL, a secret scan and a dependency audit.

See [docs/deployment.md](docs/deployment.md) for the deployment security checklist.

## Handling secrets

Never commit `.env` files, API keys or tokens. Secrets belong in the environment or a secret manager. If a secret is ever committed or pasted somewhere public, revoke it first, then remove it.

# Mem0 Self-Hosted Server

Mem0 ships a self-hosted FastAPI server plus a local dashboard. It is secure by default, supports dashboard login and API keys, and exposes OpenAPI docs at `/docs`.

## Quick Start

### Agent-first

Run one command; the terminal prints the admin email, password, and first API key.

```bash
cd server
make bootstrap
```

This starts the stack, waits for the API and dashboard to be ready, creates the first admin, and generates the first API key.

> The generated credentials print once in the `=== Ready ===` block. Save the password and API key before closing the terminal — the API key cannot be recovered afterwards.

> `make bootstrap` skips the setup wizard, so the use-case → custom-instructions step doesn't run. To add custom instructions afterwards, `POST /configure` with `{"custom_instructions": "..."}`, or run the Browser-first flow on a fresh install.

You can override the generated credentials:

```bash
cd server
make bootstrap EMAIL=admin@company.com PASSWORD='strong-password' NAME='Admin'
```

For machine-readable output:

```bash
cd server
OUTPUT=json make seed
```

Teardown:

```bash
# Stop the stack
cd server && make down

# Wipe all data (including the Postgres volume)
cd server && make clean
```

### Browser-first

Start the stack and finish setup by walking through the wizard in your browser.

```bash
cd server
make up
```

Then open `http://localhost:3000` and complete the setup wizard.

## Security Defaults

- Dashboard login uses JWTs.
- Programmatic access uses `X-API-Key`.
- Auth is enabled by default.
- `AUTH_DISABLED=true` exists for local development only and should not be used in production.

## Forgotten password

Reset an admin password from the host while the stack is running:

```bash
cd server
make reset-admin-password EMAIL=admin@example.com PASSWORD='new-strong-password'
```

This is the supported recovery path. Anyone with shell access to the host already has full access to the database and secrets, so this command does not expand the attack surface.

## Request log retention

The `request_logs` table is append-only and grows with traffic (~864k rows/day at 10 req/s). Prune it periodically:

```bash
cd server
make prune-logs                               # defaults to 30 days
make prune-logs REQUEST_LOG_RETENTION_DAYS=7  # shorter window
```

Wire the command into cron or a systemd timer in production. The `created_at` column uses a BRIN index, so range deletes stay cheap even on large tables.

## Local URLs

- Dashboard: `http://localhost:3000`
- API: `http://localhost:8888`
- OpenAPI docs: `http://localhost:8888/docs`

## Dashboard

Once logged in, the dashboard exposes:

- **Requests** — live audit log of API calls (method, path, status, latency).
- **Memories** — browse memories, filter by user ID.
- **Entities** — list every `user_id`, `agent_id`, and `run_id` that owns memories, with counts. Delete an entity to cascade-delete its memories.
- **API Keys** — create, label, and revoke per-user keys.
- **Configuration** — runtime LLM and embedder override. Changes persist to the app database and reapply on restart, layered over the values from your `.env`.
- **Settings** — account profile and password.

## Telemetry

Enabled by default, matching the Mem0 OSS library. Sends at most two events per install to the same anonymous PostHog project the library uses:

- `admin_registered` — fired when the first admin is created (wizard or direct API call). Properties: email domain, server version, install UUID.
- `onboarding_completed` — fired when the setup wizard reaches its final success state. Carries the same properties plus the freeform `use_case` the operator entered. API-only bootstraps never emit this event.

Set `MEM0_TELEMETRY=false` to opt out.

## Security headers

The dashboard sets the following response headers on every path (see `server/dashboard/next.config.mjs`):

- `X-Frame-Options: DENY`
- `Content-Security-Policy: frame-ancestors 'none'`
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: strict-origin-when-cross-origin`

Together these prevent iframe embedding, sniffing of mislabelled MIME types, and cross-origin referrer leaks. Harden further behind your own reverse proxy if needed.

## Deployment Notes

This section records the gotchas hit when wiring self-hosted Mem0 into non-default environments (custom LLM endpoints, LAN access, custom embedder dimensions). Read this before deploying anywhere that isn't `localhost`.

### 1. Custom LLM / embedder endpoints

The bundled providers list in `BUNDLED_LLM_PROVIDERS` / `BUNDLED_EMBEDDER_PROVIDERS` in `server/main.py` is restrictive (openai / anthropic / gemini). For anything else, reuse the `openai` provider — it accepts an `openai_base_url` field that points to any OpenAI-compatible gateway (vLLM, OneAPI, internal proxies).

`server/.env` template:

```env
# LLM endpoint (must be OpenAI-compatible)
OPENAI_API_KEY=sk-...
OPENAI_BASE_URL=https://your-llm-gateway.example.com/v1

# Embedder endpoint (independent from LLM if needed).
# Falls back to OPENAI_API_KEY with no base_url if left blank.
EMBEDDER_API_KEY=sk-...
EMBEDDER_BASE_URL=https://your-embedder.example.com/v1
```

`server/main.py` already reads all four and threads them into the default config.

### 2. Embedding dimensions must match the vector collection

mem0 does **not** infer the vector dimension from the embedder model. `PGVectorConfig.embedding_model_dims` defaults to `1536`, so without an explicit value the table is created with the wrong shape and every insert fails with `expected 1536 dimensions, not 1024`.

```env
# Output dimensions of the embedder model
# bge-m3 = 1024, text-embedding-3-small = 1536, text-embedding-3-large = 3072
MEM0_EMBEDDING_DIMS=1024
```

This value is wired into `vector_store.config.embedding_model_dims` in `server/main.py`. If the collection already exists with a mismatched dimension, either rename the collection (`POSTGRES_COLLECTION_NAME=new_name`) or `ALTER TABLE memories ALTER COLUMN embedding TYPE vector(<new_dim>)` in pg.

### 3. Accessing the dashboard from a non-localhost origin

CORS allowlist in `server/main.py` defaults to `["http://localhost:3000"]`. For LAN or remote access:

```env
# Comma-separated. Localhost defaults are always appended.
DASHBOARD_URL=https://dashboard.your-domain.com,http://192.168.1.165:3000
```

**Pitfall — `server/docker-compose.yaml` overrides `DASHBOARD_URL`.** The mem0 service has `DASHBOARD_URL=http://localhost:3000` in its `environment:` block, which silently overrides the value from `.env`. Change it to:

```yaml
- DASHBOARD_URL=${DASHBOARD_URL}
```

Same pattern for the dashboard's `NEXT_PUBLIC_API_URL` (this one is build-time inlined into the Next.js bundle, so a rebuild is required after changing it).

### 4. HTTP vs HTTPS — Secure cookies

The Next.js refresh-token cookie is marked `Secure` whenever the bundle was built with `NODE_ENV=production` (which is the default in `server/dashboard/Dockerfile`). `Secure` cookies are only sent over HTTPS, so a plain-HTTP LAN deployment ends up with the refresh token stranded in the cookie jar and every `/api/auth/refresh` returns 401.

**Dev / LAN fix** — `server/dashboard/src/app/api/auth/refresh/route.ts`:

```ts
const COOKIE_OPTIONS = {
  httpOnly: true,
  // dev / LAN setups use plain HTTP; flip back for HTTPS production
  secure: false,
  sameSite: "lax" as const,
  // ...
};
```

Rebuild the dashboard image (`docker compose up -d --build mem0-dashboard`) after this change.

### 5. Auth is enabled by default

`AUTH_DISABLED=true` is a development-only escape hatch. A fresh server with no `ADMIN_API_KEY` set will 401 every protected endpoint until you either:

- Set `ADMIN_API_KEY=<random>` in `.env` and create the first admin via `POST /auth/register`, or
- Set `AUTH_DISABLED=true` (local dev only), or
- Visit the `/setup` wizard in the dashboard.

`JWT_SECRET` is also mandatory when auth is enabled — `openssl rand -base64 48`. These values cannot be recovered from the running container; store them in your secret manager before closing the terminal.

### 6. Pinned toolchain versions

`server/dashboard/Dockerfile` activates pnpm via `corepack`. Without an explicit version, corepack pulls the latest pnpm, which on Node 20 ships pnpm 11.x and fails with `ERR_UNKNOWN_BUILTIN_MODULE: node:sqlite`. The fix:

```dockerfile
elif [ -f pnpm-lock.yaml ]; then corepack prepare pnpm@10.5.2 --activate && pnpm i; \
```

pnpm 10.5.2 matches `pnpm-lock.yaml` and runs on Node 20. Do not let corepack auto-pull latest without first bumping the base image to Node 22.13+.

### 7. `docker compose` does not re-read `.env` on `restart`

`docker compose restart <service>` keeps the existing container, so env-file changes are ignored. To pick up new env values: `docker compose up -d --force-recreate <service>`. For source-code changes inside a volume-mounted `dev.Dockerfile` container: a `restart` is enough since the bind mount takes effect.

## Reference

Additional product and API documentation lives at [docs.mem0.ai](https://docs.mem0.ai/open-source/overview).

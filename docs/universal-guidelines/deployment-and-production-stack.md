# Deployment and production stack

How the live demo is hosted, how to deploy it, and the failures that are
easy to hit. Authoritative for anything deployment-related; the CLAUDE.md
files carry only the rules you must not get wrong and point here.

Historical context — why this topology was chosen over the alternatives —
lives in `../plans/2026-08-15-deployment-vercel-render-supabase.md`. That
plan is a frozen snapshot; **this** file is the living reference.

## Topology

| Layer | Host | Region | Plan | URL |
|---|---|---|---|---|
| Frontend | Vercel | edge | Hobby | `https://asima-frontend-tawny.vercel.app` |
| Backend | Render | Singapore | Free (0.1 CPU / 512 MB) | `https://asima-backend-1.onrender.com` |
| Database | Supabase | `ap-southeast-1` | Free | session pooler, port 5432 |
| Object storage | Supabase Storage | `ap-southeast-1` | Free | bucket `asima`, private |

Supabase project ref: `rvdttajuhpltsgrrxrjp`. Postgres 17.6 (local dev runs
16 — a version skew worth knowing, though all 14 migrations applied cleanly).

Backend and database are both in Singapore **deliberately**. Neither Render
nor Supabase can change a service's region after creation, and a cross-region
split costs 50–60 ms *per query* — a request issuing 8–10 queries silently
adds half a second, on top of the free tier's cold start.

### Why the backend is not on Vercel

Vercel Functions cap request **and** response bodies at 4.5 MB. Leave
attachments (medical certificates) are proxied through authenticated
endpoints, so both directions cross that cap. Vercel's prescribed workaround
is client-direct presigned uploads, which would change the security model:
the bucket is private precisely so every byte passes an authenticated
endpoint. Render has no such cap.

### The Data API is disabled

The Supabase project was created with **Enable Data API off**. Without it,
PostgREST does not expose the `public` schema at all, so the app's 14 tables
are unreachable from the internet regardless of grants or RLS — which matters
because TypeORM-managed tables carry no RLS policies. The backend connects
directly over the session pooler and is unaffected.

Storage is a separate service; disabling the Data API does not affect the
S3-compatible endpoint. The dashboard's Table Editor and SQL Editor also keep
working, since Studio connects directly to Postgres.

## Environment variables

The config validators (`src/config/app.config.ts`,
`src/database/config/database.config.ts`, `src/auth/config/auth.config.ts`,
`src/storage/config/storage.config.ts`) are the source of truth. Two
categories cause different failures:

**Required — the app refuses to boot without them.** `DATABASE_TYPE`,
`DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_USERNAME`, `DATABASE_PASSWORD`,
`DATABASE_NAME`, all four `AUTH_*`, and `STORAGE_REGION`, `STORAGE_BUCKET`,
`STORAGE_ACCESS_KEY`, `STORAGE_SECRET_KEY`. You get a loud validation error
naming the offender.

**Optional in code, mandatory in production — omitting them fails silently
or confusingly:**

| Var | Value | What breaks if omitted |
|---|---|---|
| `APP_PORT` | `10000` | Defaults to 3000. Render waits on 10000; deploy hangs, then fails. |
| `DATABASE_SSL_ENABLED` | `true` | Defaults off. Supabase's pooler requires TLS — every connection fails. |
| `STORAGE_ENDPOINT` | `https://<ref>.storage.supabase.co/storage/v1/s3` | Defaults to undefined, so the AWS SDK resolves the **real AWS S3** regional endpoint and every upload fails on auth. |
| `STORAGE_FORCE_PATH_STYLE` | `true` | Virtual-host addressing against Supabase fails. |
| `STORAGE_MAX_FILE_MB` | `5` | Defaults to 10. `sharp` renders three variants per upload inside 512 MB. |

`AUTH_JWT_TOKEN_EXPIRES_IN` and `AUTH_REFRESH_TOKEN_EXPIRES_IN` are
format-checked against `/^\d+\s*[smhd]?$/` — `15m` and `7d` are valid,
`1w` is not (see the comment in `auth.config.ts` for why).

**`THROTTLE_DISABLED` must never be set in production.** `app.module.ts:63`
raises the rate limit from 300/min to 100,000/min when it is `true`. It
exists for e2e and CI only.

`FRONTEND_DOMAIN` is declared in config but **read nowhere in `src`**. Only
`CORS_ALLOWED_ORIGINS` has any effect.

### Frontend

Only two, both `NEXT_PUBLIC_*` and therefore **baked at build time** —
changing either requires a redeploy, not a restart:

```
NEXT_PUBLIC_API_BASE_URL=https://asima-backend-1.onrender.com/api/v1
NEXT_PUBLIC_APP_NAME=asima
```

Set them for **every** environment. Scoping to Production only leaves Preview
builds without them, which now fails the build (see below).

## Deploy runbook

Both apps auto-deploy on push to `main`. Order matters on first setup:

1. **Backend first**, with `CORS_ALLOWED_ORIGINS` set to a placeholder. Render
   boots fine with a wrong origin; Vercel's build hard-fails without the real
   API URL, so it cannot go first.
2. **Frontend**, with `NEXT_PUBLIC_API_BASE_URL` pointing at the live backend.
3. **Back to the backend**: replace the placeholder with the real Vercel
   origin and redeploy. Env changes do not apply until the service restarts.

Render build and start commands:

```
Build:  npm ci --include=dev && npm run build
Start:  npm run start:prod
```

Neither is the obvious default, and both matter — see gotchas 1 and 2.

## Gotchas

Every one of these was hit for real during the first deployment.

**1. `NODE_ENV=production` makes npm omit devDependencies.** npm reads it at
*install* time, not just runtime, and sets `omit=dev`. `husky` (the `prepare`
script) and `nest` both vanish, producing `npm error code 127` then
`sh: 1: nest: not found`. Fix: `--include=dev` in the build command. The
`prepare` script is now `husky || true` (`asima-backend` ac122ef) so a
production install can never be failed by a git-hook tool.

**2. `npm run start` is wrong for production.** It maps to `nest start`, which
recompiles TypeScript on every boot using a devDependency. On a free instance
that spins down after 15 minutes idle, every cold start would pay a full
compile inside 512 MB. Use `start:prod` (`node dist/main.js`).

**3. Render injects `PORT`; the app reads `APP_PORT`** (`app.config.ts:51`).
Verified by booting with `PORT=3000` and `APP_PORT=10000` — the process bound
10000 only. The boot log line to confirm is
`asima-backend listening on http://localhost:10000/api/v1`.

**4. CORS is exact string matching** (`main.ts:21-23` splits on commas, no
wildcards). Verified against a live production boot — all of these are
**rejected**:

- a trailing slash (`https://app.vercel.app/`) — and Vercel's dashboard
  displays the URL *with* one
- a path (`https://app.vercel.app/login`) — browsers never send a path in
  `Origin`
- the `http://` variant
- Vercel preview hostnames, which change every push

`CORS_ALLOWED_ORIGINS` is comma-separated if you need more than one.

**5. Duplicate Render services make routing flap.** A leftover service caused
~50% of requests to return 404 with `x-render-routing: no-server`, never
reaching the container — no `rndr-id`, no `X-Request-ID`, no Helmet headers,
plaintext `Not Found` instead of JSON — while the rest returned a clean 200.
Deleting the orphan took success from 42% to 100%. Note that `no-server` is
Render's generic reply for **any** unrecognised hostname, so probing a
hostname proves nothing either way; check the dashboard for duplicates.

**6. A missing `NEXT_PUBLIC_API_BASE_URL` used to build successfully.**
`lib/env.ts` documents its validation as a hard error, but nothing in app code
calls `env()` — `lib/api-client.ts` reads `process.env` directly with a
`http://localhost:3000/api/v1` fallback. A production build without the
variable shipped that fallback to browsers, pointing every visitor at their
own machine. `next.config.ts` now fails the build instead
(`asima-frontend` d40bf42). If a Vercel build fails with
"NEXT_PUBLIC_API_BASE_URL is required", that guard is working.

## Demo data reset

> ⚠️ **NEVER run `npm run db:fresh` or `npm run schema:drop` against
> Supabase.** `schema:drop` is not scoped to app tables, and the same database
> holds Supabase's platform-owned `auth`, `storage`, `realtime`, and `vault`
> schemas.

The seeds are all **insert-if-missing**, so re-seeding a dirty database is a
no-op — data must be cleared first. The safe sequence, implemented in
`asima-backend/.github/workflows/demo-reset.yml` (weekly cron +
`workflow_dispatch`):

1. `TRUNCATE` an explicit **14-table** list with `RESTART IDENTITY CASCADE`
2. `npm run purge:attachments:prod` — deletes everything under
   `leave-attachments/` (refuses to run without an explicit `STORAGE_BUCKET`)
3. `npm run seed:prod` — the leave-request seed re-uploads its own
   placeholders, so purging is safe

Row counts are logged before and after via `.github/scripts/demo-row-counts.sh`
so a partial failure is visible in the workflow log. A healthy reset returns
to `32/6/43/116/11/93/93` and 33 storage objects (11 attachments × 3
renditions).

Note the seed never touches existing users, so changing
`SEED_DEFAULT_PASSWORD` does not rotate live demo passwords — the truncate is
what makes that possible.

## Credentials

| File | Points at | Loaded | Committed |
|---|---|---|---|
| `asima-backend/.env` | local Postgres + MinIO | automatically, every npm script | no |
| `asima-backend/.env.supabase.prod` | live Supabase | only when explicitly sourced | no (`.env.*.prod`) |

Named `.prod`, not `.local`: `.local` is the dotenv convention for "private to
this machine", but on a file holding production credentials it reads as "the
safe local-dev one" — the misreading that ends with `db:fresh` pointed at the
live database.

To run a one-off operational task against production without editing any file:

```bash
cd asima-backend
set -a; . ./.env.supabase.prod; set +a
npm run migration:run    # or: npm run seed
```

Shell variables take precedence over dotenv, so this overrides `.env` for one
shell session and nothing on disk changes.

**Supabase S3 access keys bypass RLS and grant full access to every bucket in
the project.** They are server-only: Render env vars and GitHub Actions
secrets, never the frontend. This is a second reason the frontend proxies
attachments through the authenticated API rather than talking to storage
directly.

Demo login credentials are effectively public — all 32 seeded accounts share
`SEED_DEFAULT_PASSWORD`. Keep it demo-only and never reuse a real password.

## Verification

```bash
B=https://asima-backend-1.onrender.com

# health — `database: up` proves the pooler round-trip, not just that the app booted
curl -s $B/api/v1/health                      # {"status":"ok","database":"up"}

# auth gates
curl -s -o /dev/null -w '%{http_code}\n' $B/api/v1/users/me   # 401
curl -s -o /dev/null -w '%{http_code}\n' $B/docs              # 404 — Swagger off in prod

# CORS: only the configured origin gets an Access-Control-Allow-Origin header
curl -s -D- -o /dev/null -X OPTIONS $B/api/v1/auth/login \
  -H 'Origin: https://asima-frontend-tawny.vercel.app' \
  -H 'Access-Control-Request-Method: POST' | grep -i access-control-allow-origin
```

To confirm a frontend build baked in the right API URL, grep the deployed
chunk rather than trusting the dashboard — it should contain the Render URL
and **zero** occurrences of `localhost`.

## Keeping it awake

Two independent idle timers: Render spins a free service down after 15 minutes
(~1 minute to wake), and Supabase pauses a free project after 7 days, which
kills the database *and* storage. A single UptimeRobot monitor on
`/api/v1/health` at a 5-minute interval defeats both — the endpoint is
`@Public()`, `@SkipThrottle()`, and runs `SELECT 1`, so it exercises the
database connection rather than just the HTTP layer.

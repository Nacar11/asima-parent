# Deployment: Vercel + Render + Supabase (portfolio demo)

**Date:** 2026-08-15
**Scope:** First public deployment of all three repos, targeting a $0/month
portfolio demo — not production.
**Status:** **Phase 0 complete.** Supabase project `rvdttajuhpltsgrrxrjp` is
live in `ap-southeast-1` (Singapore, Postgres 17.6) with the Data API disabled,
all 14 migrations applied, the demo data seeded to counts identical to the local
rehearsal (`32/6/43/116/11/93/93`), and a private `asima` bucket holding the 33
attachment objects. Phase 5 is implemented (`asima-backend` 487f5a6) and
rehearsed. Phases 1–4 (Render, Vercel, attachments, keep-alive) are next.

**Pre-flight (done):** the production artifacts for Phases 1–2 were rehearsed
locally — production-mode backend boot (`APP_PORT` correctly beats Render's
injected `PORT`; health, auth, and a 404 on `/docs` all as specified), CORS
allow-list probing, and a Vercel-style frontend build (18/18 routes static).
Two findings are folded in below: the **trailing-slash CORS trap** (Phase 2.3)
and a silent-failure landmine fixed in `asima-frontend` d40bf42 — a production
build with `NEXT_PUBLIC_API_BASE_URL` unset used to succeed while baking
`http://localhost:3000/api/v1` into the client bundle; it now fails the build.

---

## Overview

Deploy `asima-frontend` to Vercel, `asima-backend` to Render's free tier, and
put both Postgres and object storage on Supabase. A free UptimeRobot monitor
keeps the two idle timers from killing the demo.

Deploying requires **no application code changes** (Phases 0–4). Every
environment difference the app cares about is already expressed as an
environment variable — `S3Storage`
(`asima-backend/src/storage/s3-storage.service.ts`) has no `if (env)` branch,
`main.ts:120` already binds `0.0.0.0`, and the build/start scripts Render
needs (`build`, `start:prod`) already exist in `package.json`.

Phase 5 (scheduled demo-data reset) adds deployment *tooling* — a purge
script and a workflow — but still changes nothing in the running app.

---

## Decision: why the backend does NOT go on Vercel

Recorded here because "put everything on Vercel" is the obvious first
instinct and it fails for a specific, non-obvious reason.

**Vercel Functions cap payloads at 4.5 MB in both directions** — request body
*and* response body ([Vercel Functions Limits][vercel-limits], updated
2026-07-01). This backend is a byte proxy by design:

- `me-leave-requests.controller.ts:62` uses `FileInterceptor('file')`, so the
  entire multipart upload lands in function memory. With
  `STORAGE_MAX_FILE_MB=10`, every attachment over 4.5 MB fails with
  `413 FUNCTION_PAYLOAD_TOO_LARGE`.
- `leave-requests.controller.ts:78` returns a `StreamableFile`. Streaming
  responses do bypass the response cap, but only via a hand-rolled
  Nest→Express serverless adapter we would have to write and maintain.
  Uploads have no equivalent escape hatch.

Vercel's own guidance for this limit is to treat functions as "a lightweight
API layer, not a media server" and move to client-direct presigned uploads.
**That is incompatible with the deliberate design here.** The bucket is
private and every byte crosses an authenticated endpoint because attachments
are medical certificates — `leave-attachment.tsx:12` documents it: bytes are
fetched with the caller's bearer token, "never an unauthenticated
`<img src>`". Adopting Vercel would mean changing the security model to suit
a hosting constraint.

Secondary costs of the Vercel path, for the record:

| Concern | On Vercel | On Render |
|---|---|---|
| Entry point | `main.ts:15` calls `app.listen()` — needs a new `api/index.ts`, `vercel.json`, cached bootstrap | `npm run start:prod`, already present |
| Cold boot | 13 modules + 13 entities + glob entity scan (`typeorm-config.service.ts:22`) every cold start | Once, at deploy |
| DB pool | `DATABASE_MAX_CONNECTIONS=20` per instance × autoscale — must drop to 1 + transaction pooler | Works as written |
| Rate limiting | Throttler is in-memory (`app.module.ts:59`); 300/min becomes per-instance fiction | Works as written |
| Migrations | No release step | Run before deploy |
| Code changes | Adapter + config + upload redesign | **None** |

**Decision: Render free tier for the API.** The ~1 minute cold start is
accepted (decided 2026-08-15) — the UptimeRobot monitor in Phase 4 keeps the
service warm in practice, so the wake penalty should only ever be paid if the
monitor itself is down. Render Starter ($7/mo) remains the escape hatch if
that turns out to be optimistic; it removes the spin-down with no other
changes.

---

## Target topology

```
                    ┌──────────────────────────┐
  browser  ────────►│  Vercel (Hobby)          │
                    │  asima-frontend, Next 15 │
                    └───────────┬──────────────┘
                                │  NEXT_PUBLIC_API_BASE_URL
                                │  (baked at build time)
                                ▼
                    ┌──────────────────────────┐
                    │  Render (Free)           │
                    │  asima-backend, NestJS   │
                    │  512 MB RAM / 0.1 CPU    │
                    └───────┬──────────┬───────┘
                            │          │  S3 API (private bucket)
              Postgres (SSL)│          │
                            ▼          ▼
                    ┌──────────────────────────┐
                    │  Supabase (Free)         │
                    │  Postgres  +  Storage    │
                    └──────────────────────────┘

  UptimeRobot ──► GET /api/v1/health every 5 min
                  (wakes Render, keeps Supabase from pausing)
```

Note the browser never talks to Supabase Storage directly — attachment bytes
always flow bucket → Render → browser through an authenticated endpoint. No
bucket CORS configuration is needed.

---

## Phase 0: Supabase foundation

### Task 0.1: Create the Supabase project and capture connection details

**Description:** Provision the free-tier project that will host both Postgres
and the attachment bucket.

**Acceptance criteria:**
- [ ] Project created; region noted (used later for `STORAGE_REGION`)
- [ ] Database password saved to a password manager
- [ ] **Session pooler** connection string copied (not the direct connection —
      it is IPv6-only without the IPv4 add-on, and Render needs IPv4)

**Verification:**
- [ ] `psql "postgresql://postgres.<ref>:<pw>@aws-0-<region>.pooler.supabase.com:5432/postgres" -c 'select 1'` returns a row

**Dependencies:** None
**Scope:** XS (no files)

---

### Task 0.2: Create the private storage bucket and S3 credentials

**Description:** Create the `asima` bucket and generate S3-protocol
credentials so the existing `S3Storage` adapter can talk to it unchanged.

**Acceptance criteria:**
- [ ] Bucket named `asima` exists and is **private** (public toggle OFF)
- [ ] S3 access key ID + secret generated under Storage → S3 access keys
- [ ] Endpoint recorded as `https://<project-ref>.supabase.co/storage/v1/s3`

**Verification:**
- [ ] From the backend repo with `STORAGE_*` pointed at Supabase, run the
      storage unit tests: `npm run test -- storage`
- [ ] Manual: bucket is not browsable anonymously

**Dependencies:** 0.1
**Scope:** XS (no files — env only)

---

### Task 0.3: Migrate and seed the Supabase database

**Description:** Run the 14 existing migrations and the seed against Supabase
from your local machine. Doing this locally avoids depending on Render's
pre-deploy command, whose availability varies by plan.

**Acceptance criteria:**
- [ ] All 14 migrations applied
- [ ] Seed data present (roles, permissions, demo users)
- [ ] Local `.env` reverted to local Postgres afterward

**Verification:**
```bash
# In asima-backend, temporarily point .env at Supabase with
# DATABASE_SSL_ENABLED=true, then:
npm run migration:run
npm run seed
psql "$SUPABASE_URL" -c '\dt'          # 14 tables + migrations ledger
psql "$SUPABASE_URL" -c 'select count(*) from users'
```

**Dependencies:** 0.1
**Scope:** XS (no files — `.env` only, and reverted)

> **Careful:** `db:fresh` runs `schema:drop` first. Never point it at
> Supabase. Use `migration:run` + `seed` only.

---

### ✅ Checkpoint: Foundation

- [ ] Supabase reachable over the session pooler with SSL
- [ ] Schema and seed data in place
- [ ] Private bucket with working S3 credentials
- [ ] Local `.env` restored — `git status` clean in `asima-backend`

---

## Phase 1: Backend on Render

Highest-risk phase; done early so failures surface before anything depends
on them.

### Task 1.1: Create the Render web service

**Description:** Point Render at `Nacar11/asima-backend` and configure the
Node build. No Dockerfile needed — the backend has none, and Render's Node
runtime handles it.

**Acceptance criteria:**
- [ ] Repo connected, branch `main`, auto-deploy on push
- [ ] Runtime: Node (>= 20, per `package.json` engines)
- [ ] Build command: `npm ci && npm run build`
- [ ] Start command: `npm run start:prod`
- [ ] Instance type: Free

**Verification:**
- [ ] Build log shows `nest build` succeeding and `dist/main.js` produced

**Dependencies:** 0.3
**Scope:** XS (no files)

---

### Task 1.2: Set backend environment variables

**Description:** Populate the full env set. See the reference table below.
The port is the one non-obvious item: Render injects `PORT`, but
`app.config.ts:53` reads **`APP_PORT`**, so it must be set explicitly.

**Acceptance criteria:**
- [ ] Every variable in the backend table below is set
- [ ] `APP_PORT=10000` (matches Render's default expected port)
- [ ] `DATABASE_SSL_ENABLED=true`
- [ ] JWT secrets are freshly generated, and access ≠ refresh
- [ ] `THROTTLE_DISABLED` is **not set** (it exists for e2e only)
- [ ] `CORS_ALLOWED_ORIGINS` holds a placeholder for now — the real Vercel
      URL does not exist yet (resolved in Task 2.3)

**Verification:**
- [ ] Service starts; logs show `asima-backend listening on ...`
- [ ] `curl https://<service>.onrender.com/api/v1/health` → `{"status":"ok","database":"up"}`

**Dependencies:** 1.1
**Scope:** XS (no files)

---

### ✅ Checkpoint: API alive

- [ ] `/api/v1/health` returns 200 with `database: up`
- [ ] A protected route returns 401 without a token (guards are wired)
- [ ] `POST /api/v1/auth/login` with a seeded user returns tokens
- [ ] Swagger is **not** exposed (`main.ts:41` gates it on `NODE_ENV !== 'production'`)

---

## Phase 2: Frontend on Vercel

### Task 2.1: Create the Vercel project

**Description:** Import `Nacar11/asima-frontend` as its own Vercel project.
It is a standalone repo, so no monorepo root-directory configuration is
needed.

**Acceptance criteria:**
- [ ] Project imported, framework auto-detected as Next.js
- [ ] Node 20+ selected
- [ ] Build succeeds

**Verification:**
- [ ] Deployment URL loads the login page

**Dependencies:** None (can run in parallel with Phase 1)
**Scope:** XS (no files)

> `next.config.ts` sets `output: 'standalone'`. That exists for
> `asima-frontend/docker/Dockerfile` and is a self-hosting setting — Vercel
> builds Next natively and does not need it. **Do not delete it** (the Docker
> image depends on it). If the Vercel build ever behaves oddly, this is the
> first knob to investigate.

---

### Task 2.2: Set frontend environment variables

**Acceptance criteria:**
- [ ] `NEXT_PUBLIC_API_BASE_URL=https://<service>.onrender.com/api/v1`
- [ ] `NEXT_PUBLIC_APP_NAME=asima`
- [ ] Redeployed after setting them

**Verification:**
- [ ] Browser devtools shows API calls going to the Render host, not localhost

> Both are `NEXT_PUBLIC_*`, so they are **baked at build time** — changing
> either requires a redeploy, not just a restart. Make sure
> `NEXT_PUBLIC_API_BASE_URL` is enabled for the environment being built
> (setting it for Production only leaves Preview builds without it).
>
> Since `asima-frontend` d40bf42 a production build **fails** when it is
> missing or malformed. Before that fix the build succeeded and shipped a
> `http://localhost:3000/api/v1` fallback to the browser, which points at the
> *visitor's* machine — a green deploy where nothing works. If the Vercel
> build fails with "NEXT_PUBLIC_API_BASE_URL is required", that guard is
> doing its job: set the variable and redeploy.

**Dependencies:** 1.2, 2.1
**Scope:** XS (no files)

> Both are `NEXT_PUBLIC_*`, so they are **baked into the client bundle at
> build time**. Changing either requires a redeploy, not just an env edit.

---

### Task 2.3: Close the CORS loop

**Description:** The classic chicken-and-egg: the backend needs the Vercel
URL, which does not exist until the frontend deploys. Resolve it here.

**Acceptance criteria:**
- [ ] Backend `CORS_ALLOWED_ORIGINS` = the real Vercel production URL
- [ ] Backend `FRONTEND_DOMAIN` = same URL
- [ ] Backend redeployed

**Verification:**
- [ ] Login from the Vercel URL succeeds with no CORS error in the console
- [ ] `curl -H 'Origin: https://evil.example' -I https://<service>.onrender.com/api/v1/health`
      does **not** return a permissive `Access-Control-Allow-Origin`

**Dependencies:** 2.2
**Scope:** XS (no files)

> `main.ts:22` splits `CORS_ALLOWED_ORIGINS` on commas and matches exactly —
> **no wildcards**. Vercel preview deployments get random hostnames that will
> therefore fail CORS. Either accept that only production works, or add
> specific preview URLs as needed.
>
> **Verified locally against a production-mode boot.** Only the exactly
> configured origin received an `Access-Control-Allow-Origin` header. Blocked:
> preview hostnames, the `http://` variant, unrelated origins — **and the same
> URL with a trailing slash**. That last one is the trap: Vercel's dashboard
> displays the deployment URL *with* a trailing slash, so pasting it verbatim
> here produces a green deploy where every login fails CORS. Enter the origin
> with no trailing slash and no path.

---

### ✅ Checkpoint: End-to-end login

- [ ] Log in from the Vercel URL with a seeded user
- [ ] Token refresh works (leave the tab open past 15 minutes, or force it)
- [ ] Logout revokes refresh tokens (per ADR 0002)
- [ ] A permission-gated admin route renders correctly for an admin user

---

## Phase 3: The attachment vertical

The storage path is the one that most deserves its own end-to-end check,
because it is the only path touching all three providers at once.

### Task 3.1: Verify the attachment round-trip

**Description:** Exercise upload → sharp derivation → storage → authenticated
download against real infrastructure.

**Acceptance criteria:**
- [ ] Submitting a sick-leave request with a JPEG attachment succeeds
- [ ] Thumbnail renders in the leave detail drawer
- [ ] "View full image" opens the original
- [ ] A PDF attachment falls back to the download link (no thumbnail)
- [ ] Objects appear in the Supabase bucket under the
      `leave-attachments/` prefix

**Verification:**
- [ ] Supabase Storage browser shows `original`, `preview`, `thumbnail`
      renditions for the image
- [ ] Fetching an object URL anonymously fails (bucket is private)

**Dependencies:** 0.2, 2.3
**Scope:** XS (no files — manual verification)

---

### Task 3.2: Tune the file size cap for the free tier

**Description:** Decide the demo's upload ceiling against 512 MB of RAM.

**Acceptance criteria:**
- [ ] `STORAGE_MAX_FILE_MB` set deliberately (recommended: `5` for the demo)

**Rationale:** multer buffers the whole upload in memory, then
`image-processor.service.ts:32` derives preview and thumbnail **in parallel**
via sharp, which allows a decoded canvas up to 24 megapixels. A 24 MP decode
is roughly 96 MB of bitmap, and two run concurrently. On a 512 MB instance
with a 10 MB source buffer also resident, headroom is thin. Lowering the cap
to 5 MB for the demo is cheaper than debugging an OOM restart mid-interview.

**Verification:**
- [ ] Upload a ~4 MB photo — succeeds
- [ ] Upload a file over the cap — clean 422 with the size message, no crash
- [ ] Render logs show no OOM restart

**Dependencies:** 3.1
**Scope:** XS (no files)

---

## Phase 4: Keep the demo alive

### Task 4.1: Add the UptimeRobot monitor

**Description:** One monitor defeats both idle timers. `/api/v1/health` is
`@Public()`, `@SkipThrottle()`, and runs `SELECT 1`
(`health.controller.ts:26`), so each ping wakes Render *and* registers as
Supabase database activity.

**Acceptance criteria:**
- [ ] HTTP(s) monitor on `https://<service>.onrender.com/api/v1/health`
- [ ] Interval: 5 minutes
- [ ] Alert contact configured (email is fine)

**Verification:**
- [ ] Monitor green for 24 hours
- [ ] Loading the Vercel URL cold responds immediately (no 1-minute wake)
- [ ] After 8+ days, the Supabase project is still active (not paused)

**Dependencies:** 1.2
**Scope:** XS (no files)

> Budget check: Render grants 750 free instance-hours per month per
> workspace. One continuously-awake service is ~720 h/month, which fits — but
> only for **one** free service. A second always-on free service would blow
> the allowance and suspend both.

---

### ✅ Checkpoint: Demo-ready

- [ ] Cold visit to the Vercel URL reaches a usable app in < 5 s
- [ ] Login, leave request with attachment, and approval all work
- [ ] `X-Request-ID` present on responses (correlation works)
- [ ] Demo credentials documented somewhere you can read them under pressure

---

## Phase 5: Scheduled demo-data reset

Decided 2026-08-15: demo data resets on a schedule rather than accumulating
whatever visitors click into existence.

**This phase is the one exception to "no application code changes."** It adds
deployment tooling — a workflow and a purge script — but changes nothing in
the running app.

### How reset works, and why not `db:fresh`

Two facts from the codebase determine the design:

1. **Every seed service is insert-if-missing.** `user-seed.service.ts:84`,
   `time-entry-seed.service.ts:45`, and `leave-request-seed.service.ts:110`
   all skip rows that already exist. Re-running `seed` on a dirty database
   therefore does nothing — a reset *must* clear data first.
2. **The seed uploads its own attachments.** `leave-request-seed.service.ts:124`
   calls `uploadPlaceholder(...)` for employees whose rows need one, and only
   when those rows are missing. Purging the bucket is therefore safe: the
   next seed re-uploads what it needs.

**`db:fresh` must never be used against Supabase.** It runs `schema:drop`
first, which is not scoped to this app's tables — on Supabase the same
database also holds the `auth`, `storage`, and `realtime` schemas that the
platform itself owns. Truncating an explicit table list is the safe
equivalent.

Reset sequence:

```
1. TRUNCATE the 14 app tables (RESTART IDENTITY CASCADE) — explicit list
2. Purge the leave-attachments/ prefix from the bucket
3. npm run seed:prod  → users, roles, schedules, chains, demo leave
                        requests, and freshly uploaded placeholders
```

Truncating all 14 (rather than only the transactional ones) also removes
users a visitor created through the admin UI, returning the demo to exactly
its seeded state.

---

### Task 5.1: Add the bucket purge script

**Description:** A small script that deletes every object under the
`leave-attachments/` prefix, so orphaned renditions don't accumulate against
the 1 GB free-tier ceiling across resets.

**Acceptance criteria:**
- [ ] Script lists and deletes all objects under `LEAVE_ATTACHMENT_PREFIX`
- [ ] Reuses the existing `@aws-sdk/client-s3` dependency and `STORAGE_*`
      config — no new dependency, no second storage code path
- [ ] Refuses to run unless `STORAGE_BUCKET` is set explicitly (guards
      against pointing it at the wrong bucket)
- [ ] Exposed as an npm script alongside the existing `seed` scripts

**Verification:**
- [ ] Run against a local MinIO bucket with objects present → bucket empty
- [ ] Run against an already-empty bucket → exits 0, no error

**Dependencies:** 3.1
**Files likely touched:**
- `asima-backend/src/database/seeds/` (new purge script — the seeds directory
  is where reset tooling belongs)
- `asima-backend/package.json` (one script entry)

**Scope:** S (1–2 files)

---

### Task 5.2: Add the scheduled reset workflow

**Description:** A GitHub Actions cron in `asima-backend` that runs the three
reset steps against Supabase on a schedule.

**Acceptance criteria:**
- [ ] Weekly cron (plus `workflow_dispatch` so it can be triggered by hand
      before a demo)
- [ ] Truncates via an explicit 14-table list — never `schema:drop`
- [ ] Runs the purge script, then `npm run seed:prod`
- [ ] Supabase and storage credentials read from repository secrets, never
      committed
- [ ] Logs the row counts before and after so a failed reset is visible

**Verification:**
- [ ] Trigger manually via `workflow_dispatch`; run is green
- [ ] Create a leave request in the demo, run the workflow, confirm it is
      gone and the seeded demo data is back
- [ ] Confirm the bucket holds only freshly uploaded placeholders
- [ ] Confirm login still works with `SEED_DEFAULT_PASSWORD`

**Dependencies:** 5.1, 4.1
**Files likely touched:**
- `asima-backend/.github/workflows/` (new workflow)

**Scope:** S (1 file)

> **GitHub Actions cron caveats:** scheduled workflows are best-effort and can
> be delayed by many minutes under load, and GitHub disables schedules in
> repositories with no activity for 60 days. Neither matters much for a demo
> reset — but if the schedule ever goes quiet, repository inactivity is the
> first thing to check.

---

### ✅ Checkpoint: Reset works

- [ ] Manual `workflow_dispatch` run completes green
- [ ] Demo returns to seeded state: visitor-created leave requests, time
      entries, and users all gone
- [ ] Attachments render after reset (placeholders re-uploaded)
- [ ] Login works with the demo credentials

---

## Environment variable reference

### Backend — Render

| Variable | Value | Notes |
|---|---|---|
| `NODE_ENV` | `production` | Also disables Swagger (`main.ts:41`) |
| `APP_NAME` | `asima` | |
| `APP_PORT` | `10000` | **Render injects `PORT`; the app reads `APP_PORT`** |
| `API_PREFIX` | `api` | |
| `FRONTEND_DOMAIN` | `https://<app>.vercel.app` | |
| `CORS_ALLOWED_ORIGINS` | `https://<app>.vercel.app` | Exact match, comma-separated, no wildcards |
| `DATABASE_TYPE` | `postgres` | |
| `DATABASE_HOST` | `aws-0-<region>.pooler.supabase.com` | Session pooler (IPv4) |
| `DATABASE_PORT` | `5432` | Session pooler port |
| `DATABASE_USERNAME` | `postgres.<project-ref>` | Pooler username format |
| `DATABASE_PASSWORD` | *(secret)* | |
| `DATABASE_NAME` | `postgres` | Supabase default |
| `DATABASE_SYNCHRONIZE` | `false` | Never true |
| `DATABASE_MAX_CONNECTIONS` | `10` | Down from 20 — single small instance |
| `DATABASE_SSL_ENABLED` | `true` | **Required**; drives `typeorm-config.service.ts:25` |
| `AUTH_JWT_SECRET` | *(secret)* | `openssl rand -base64 48` |
| `AUTH_JWT_TOKEN_EXPIRES_IN` | `15m` | |
| `AUTH_REFRESH_SECRET` | *(secret)* | Must differ from the access secret |
| `AUTH_REFRESH_TOKEN_EXPIRES_IN` | `7d` | |
| `STORAGE_ENDPOINT` | `https://<project-ref>.supabase.co/storage/v1/s3` | |
| `STORAGE_REGION` | *(project region)* | e.g. `ap-southeast-1` |
| `STORAGE_BUCKET` | `asima` | |
| `STORAGE_ACCESS_KEY` | *(secret)* | Supabase S3 access key ID |
| `STORAGE_SECRET_KEY` | *(secret)* | |
| `STORAGE_FORCE_PATH_STYLE` | `true` | Supabase requires path-style |
| `STORAGE_MAX_FILE_MB` | `5` | See Task 3.2 |
| `SEED_DEFAULT_PASSWORD` | *(demo password)* | Only read by the seed script |

**Not set:** `THROTTLE_DISABLED`, `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`
(local-only).

### Frontend — Vercel

| Variable | Value | Notes |
|---|---|---|
| `NEXT_PUBLIC_API_BASE_URL` | `https://<service>.onrender.com/api/v1` | Build-time baked |
| `NEXT_PUBLIC_APP_NAME` | `asima` | Build-time baked |

---

## Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Supabase pauses the project after 7 days idle — DB **and** storage go dark | **High** | UptimeRobot ping hits `/health`, which runs `SELECT 1` |
| Render sleeps after 15 min; ~1 min cold start reads as "broken" | Med | Same ping keeps it warm |
| 0.1 CPU makes sharp derivation slow on upload | Med | Demo with modest images; accept a few seconds |
| 512 MB RAM + in-memory upload + parallel sharp decode → OOM | Med | `STORAGE_MAX_FILE_MB=5` (Task 3.2) |
| Vercel preview deploys fail CORS (random hostnames) | Low | Expected; demo from the production URL |
| Supabase free caps: 500 MB DB, 1 GB storage, 5 GB egress/month | Low | Far beyond demo needs; watch egress if the demo goes viral |
| Vercel Hobby is non-commercial only | Low | A portfolio qualifies |
| Seeded demo credentials are effectively public | Med | Use a demo-only password; never reuse a real one |
| Render's 750 free instance-hours cover only one always-on service | Low | Do not add a second free always-on service |
| A scheduled reset fires mid-demo and wipes what you just showed | Med | Weekly cadence at a quiet hour; run `workflow_dispatch` manually *before* a demo rather than relying on the schedule |
| `db:fresh` / `schema:drop` pointed at Supabase would drop platform-owned schemas | **High** | Reset truncates an explicit 14-table list; `db:fresh` is never used against Supabase (Phase 5) |
| GitHub disables cron schedules after 60 days of repository inactivity | Low | Expected for a portfolio repo; check this first if resets stop |
| Orphaned bucket objects accumulate across resets | Low | Purge script runs as reset step 2 (Task 5.1) |

---

## Non-goals

- Custom domains (add later; requires updating `CORS_ALLOWED_ORIGINS`,
  `FRONTEND_DOMAIN`, and `NEXT_PUBLIC_API_BASE_URL` + redeploys)
- CI-driven deploys — both platforms auto-deploy on push to `main`
- Automated migration release step — run locally per Task 0.3
- Production hardening: the strict CSP deferred in `next.config.ts`, a shared
  throttler store, log aggregation, backups

---

## Decisions log

| Date | Question | Decision |
|---|---|---|
| 2026-08-15 | Where does the NestJS API run? | Render free tier — Vercel's 4.5 MB payload cap is incompatible with the attachment proxy design (see Decision section) |
| 2026-08-15 | Where do attachments live? | Supabase Storage over its S3-compatible endpoint — zero change to `S3Storage` |
| 2026-08-15 | Reset demo data, or let it accumulate? | **Reset on a schedule** — truncate 14 tables + purge bucket + reseed (Phase 5) |
| 2026-08-15 | Is the ~1 min cold start acceptable? | **Yes** — stay on Render free; the keep-alive monitor should make it rare. Render Starter ($7/mo) stays the escape hatch |

No open questions.

---

## Sources

Platform limits verified 2026-08-15:

- [Vercel Functions Limits][vercel-limits] — 4.5 MB payload cap, Hobby 300 s
  duration, 2 GB memory
- [Bypassing the 4.5 MB body size limit](https://vercel.com/kb/guide/how-to-bypass-vercel-body-size-limit-serverless-functions)
  — confirms the cap applies to responses; streaming is exempt
- [Vercel Limits](https://vercel.com/docs/limits) — Hobby allowances, Git
  organization restriction
- [Running Docker on Vercel vs Render](https://vercel.com/kb/guide/docker-on-vercel-vs-render)
  — Vercel's own guidance on when to choose Render
- [Deploy for Free – Render Docs](https://render.com/docs/free) — 15-minute
  spin-down, ~1 minute wake, 750 instance-hours
- [Render Instance Types](https://render.com/docs/compute-plans) — free tier
  512 MB / 0.1 CPU
- [Supabase Pricing](https://supabase.com/pricing) — 500 MB DB, 1 GB storage,
  5 GB egress; free projects pause after 7 days idle

[vercel-limits]: https://vercel.com/docs/functions/limitations

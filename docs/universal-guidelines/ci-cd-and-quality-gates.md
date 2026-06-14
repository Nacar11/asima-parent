# CI/CD and quality gates — how the three repos stay green

This guide explains how Asima keeps `main` healthy across all three repos
(`asima-parent`, `asima-backend`, `asima-frontend`). It's written to be
understood by someone who has never set up CI before — concepts first, then
the exact wiring per repo.

> **Source of truth:** the workflow files (`.github/workflows/*.yml`) and the
> hook files (`.husky/pre-push`) are authoritative. This doc *explains and
> excerpts* them. If a snippet here ever disagrees with the real file, the
> file wins — and you should fix this doc (see "When to update THIS file").

---

## TL;DR — the mental model

Quality is enforced at **two layers**, and you want both:

```
  YOU                          GITHUB
  ───                          ──────
  git commit        (free, no checks)
  git push   ──► [ pre-push hook ]  ──►  upload  ──► [ CI workflow ]
                  runs on YOUR machine                runs on a clean VM
                  BEFORE the push leaves              AFTER the push lands
                  (shift-left gate)                   (backstop / smoke alarm)
```

- **Pre-push hook (local, Husky):** runs the checks on your machine *before*
  the push is sent. If a check fails, the push is **aborted** — the bad code
  never leaves your laptop.
- **CI (GitHub Actions):** runs the same checks on a clean Linux VM *after*
  the push lands on `main`. It's the backstop that catches anything the local
  hook missed (or that someone bypassed).

**Why both?** We **commit straight to `main`** (no pull requests — see the root
`CLAUDE.md` git workflow). With no PR, CI can't *block a merge*; by the time it
runs, the code is already on `main`. So CI is a **smoke alarm**, and the
**pre-push hook is the real pre-merge gate**. Catch problems before the push.

**The golden rule of ordering:** run the cheapest checks first
(lint → format → types → tests → build). A formatting typo should fail in
10 seconds, not after a 4-minute build. This is called **shifting left**.

---

## The two gates, side by side

| | Pre-push hook | CI workflow |
|---|---|---|
| Lives in | `.husky/pre-push` (each app repo) | `.github/workflows/*.yml` (each repo) |
| Runs on | your machine | a fresh GitHub-hosted Ubuntu VM |
| Triggers | every `git push` | `push` to `main` **and** `pull_request` → `main` |
| If it fails | the push is **aborted** | the run goes **red** (✗ in the Actions tab) |
| Speed need | must be fast (you wait on it) | can be slower (runs in background) |
| Can be skipped? | yes, `git push --no-verify` (don't) | no |
| Role here | the real pre-merge gate (direct-to-main) | backstop / visibility |

Both run the *same family* of checks so there are no surprises: if it passes
locally, it passes in CI.

---

## What's enforced where

This is the quick-reference. "Blocking" = a failure stops the push / fails the
CI run. "Info" = it runs and prints output but never fails the build.

| Gate | What it does | Frontend | Backend | Parent |
|---|---|---|---|---|
| **lint** (`lint:ci`) | ESLint, no auto-fix | pre-push + CI (blocking) | pre-push + CI (blocking) | — |
| **format** (`format:check`) | Prettier `--check` | pre-push + CI (blocking) | pre-push + CI (blocking) | — |
| **typecheck** | `tsc --noEmit` | pre-push + CI (blocking) | pre-push + CI (blocking) | — |
| **unit tests** | Vitest / Jest | pre-push + CI (blocking) | pre-push + CI (blocking) | — |
| **build** | `next build` / `nest build` | CI (blocking) | CI (blocking) | — |
| **migrate + e2e** | real Postgres, then e2e specs | — | CI only (blocking) | — |
| **npm audit** | known-vuln scan | CI (**info**) | CI (**info**) | — |
| **plan naming** | `docs/plans/` filenames | — | — | CI (blocking) |
| **no committed todos** | block `*-todo.md` in `docs/plans/` | — | — | CI (blocking) |
| **link check** | offline markdown links | — | — | CI (blocking) |

Notes:
- **Build is CI-only**, not in the pre-push hook — it's slow, and the
  typecheck already catches the type errors a build would. Keeps pushes fast.
- **e2e is CI-only** because it needs a Postgres database (CI spins one up; see
  below). The pre-push hook stays fast with unit tests only.
- **npm audit is informational everywhere** — see "Key decisions" for why.

---

## Per-repo detail

Each app repo (`asima-backend`, `asima-frontend`) has the same shape: a set of
npm scripts, a CI workflow that calls them, and a Husky pre-push hook that
calls the fast subset. The parent repo is docs-only and has a different
workflow.

### Frontend — Next.js + Vitest

**Scripts** (`package.json`). The `:ci`/`:check` variants are the
**non-mutating** twins of the everyday scripts (no `--fix`/`--write`):

```jsonc
"lint":         "next lint --fix",       // local convenience: auto-fixes
"lint:ci":      "next lint",             // CI/hook: verify only
"typecheck":    "tsc --noEmit",
"format":       "prettier --write ...",  // local: rewrites files
"format:check": "prettier --check ...",  // CI/hook: fails if unformatted
"test":         "vitest run",
"build":        "next build"
```

**CI** (`.github/workflows/ci.yml`) — runs on push/PR to `main`:

```yaml
env:
  # next/font + page metadata read NEXT_PUBLIC_* at BUILD time, so the
  # build needs a value present. The real URL is set per-environment.
  NEXT_PUBLIC_API_BASE_URL: http://localhost:3000/api/v1
  NEXT_TELEMETRY_DISABLED: '1'
steps:
  - checkout → setup-node@20 (npm cache) → npm ci
  - npm run lint:ci         # blocking
  - npm run format:check    # blocking
  - npm run typecheck       # blocking
  - npm test                # blocking
  - npm run build           # blocking
  - npm audit --audit-level=high   # continue-on-error: true → INFO ONLY
```

**Pre-push hook** (`.husky/pre-push`) — the fast subset, no build:

```sh
set -e
npm run lint:ci
npm run format:check
npm run typecheck
npm test
```

### Backend — NestJS + Jest + TypeORM

**Scripts** (`package.json`):

```jsonc
"lint":         "eslint \"{src,test}/**/*.ts\" --fix",
"lint:ci":      "eslint \"{src,test}/**/*.ts\"",        // no --fix
"typecheck":    "tsc --noEmit -p tsconfig.json",        // added for parity
"format":       "prettier --write \"src/**/*.ts\" \"test/**/*.ts\"",
"format:check": "prettier --check \"src/**/*.ts\" \"test/**/*.ts\"",
"test":         "jest",
"test:e2e":     "... jest --config ./test/jest-e2e.json --runInBand",
"build":        "nest build"
```

**CI** (`.github/workflows/ci.yml`) — the richer pipeline, because e2e needs a
real database. GitHub gives the job a throwaway **Postgres service container**:

```yaml
services:
  postgres:                        # a disposable DB, alive only for this run
    image: postgres:16-alpine
    env: { POSTGRES_USER: asima, POSTGRES_PASSWORD: asima, POSTGRES_DB: asima_test }
    ports: ['5432:5432']
    options: >- --health-cmd "pg_isready ..." ...   # wait until it's ready
env:
  # CI-only throwaway config (see "CI environment & secrets" below):
  DATABASE_HOST: localhost
  AUTH_JWT_SECRET: ci-access-secret-do-not-use-in-prod
  THROTTLE_DISABLED: 'true'        # e2e logs in many times; skip rate limits
  # ...full env block in the file...
steps:
  - checkout → setup-node@20 (npm cache) → npm ci
  - npm run lint:ci          # blocking
  - npm run format:check     # blocking
  - npm run typecheck        # blocking
  - npm run build            # blocking
  - npm test -- --ci         # unit tests, blocking
  - npm run migration:run    # apply migrations to the throwaway Postgres
  - npm run test:e2e -- --ci # e2e against the real DB, blocking
  - npm audit --audit-level=high   # continue-on-error: true → INFO ONLY
```

**Pre-push hook** (`.husky/pre-push`) — fast subset; **e2e is intentionally
excluded** (it needs Postgres) and is left to CI:

```sh
set -e
npm run lint:ci
npm run format:check
npm run typecheck
npm test          # jest UNIT tests only — no e2e here
```

### Parent — docs-only

`asima-parent` has no application code, so its CI enforces the **documentation
conventions** from the root `CLAUDE.md` instead of lint/test/build.

**CI** (`.github/workflows/docs.yml`):

1. **Plan filename convention** — every file in `docs/plans/` must start with
   `YYYY-MM-DD-` (a small bash check; fails the run with a `::error` annotation
   otherwise).
2. **No committed todo snapshots** — blocks any `docs/plans/*-todo.md`. Todos
   live only in the gitignored `tasks/todo.md`; they are not part of the audit
   trail.
3. **Markdown link check** — `lycheeverse/lychee-action` run with `--offline`,
   so it validates **relative links between docs** without hitting the network
   (network checks would be flaky / rate-limited).

There is **no Husky hook** in the parent repo — it has no `package.json` and
the checks are cheap enough to live in CI alone.

---

## CI environment & secrets

This is the part people get wrong, so read it carefully.

The backend CI workflow contains real-looking values like
`AUTH_JWT_SECRET: ci-access-secret-do-not-use-in-prod` and
`DATABASE_PASSWORD: asima` **in plain text in the YAML**. That is **safe and
intentional**, because:

- They are **throwaway, CI-only** values. They authenticate against a
  **disposable** Postgres container that is created and destroyed within a
  single CI run. Nothing they protect outlives the job.
- The names even say so (`...-do-not-use-in-prod`).

**The rule that must never be broken:**

> **Real secrets never go in code or in workflow YAML.** Production database
> passwords, real JWT secrets, cloud/API keys — those live in **GitHub
> Secrets** (or the deployment platform's secret store) and are referenced as
> `${{ secrets.NAME }}`. They are never committed.

Today there are no production secrets in CI because there is no deployment
pipeline yet. When CD lands (see "Future"), the deploy job will read real
secrets from GitHub Secrets — *never* from a committed file.

Frontend note: `NEXT_PUBLIC_API_BASE_URL` is **not a secret** — anything
prefixed `NEXT_PUBLIC_` is baked into the browser bundle and is public by
design. It's set in CI only because the build reads it at build time.

---

## The standard stack

You don't choose tools per change — these are the project-wide standard. New
work uses them; adding an alternative needs an ADR.

| Layer | Tool | Notes |
|---|---|---|
| Package manager | **npm** | CI installs with `npm ci` (exact versions from `package-lock.json`). |
| Node version | **20** | CI pins `node-version: '20'`; `engines.node: ">=20"` in both apps. |
| Formatter | **Prettier** | Backend: `singleQuote`, `trailingComma: all`, `printWidth: 100`. Frontend: same + `semi`, `tabWidth: 2`, **`prettier-plugin-tailwindcss`** (sorts Tailwind classes). |
| Linter | **ESLint** | `.eslintrc.js` (backend) / `.eslintrc.json` (frontend). |
| Types | **TypeScript** | `tsc --noEmit` as the typecheck gate. |
| Tests | **Jest** (backend) / **Vitest** (frontend) | Backend also has Jest e2e (in-band, real Postgres). |
| Git hooks | **Husky v9** | `prepare: "husky"` installs hooks on `npm install`. |

**`npm ci` vs `npm install`:** CI always uses `npm ci`, which installs the
*exact* versions in `package-lock.json` and fails if `package.json` and the
lockfile disagree. That's why a lockfile change must be committed alongside a
dependency change.

---

## The pre-push hook (Husky), explained

**What Husky is:** a tiny tool that installs **git hooks** — scripts git runs
automatically at certain points. We use the **`pre-push`** hook: git runs it
right before a `git push`, and if the script exits non-zero, **git cancels the
push**.

**How it gets installed:** `package.json` has `"prepare": "husky"`. `prepare`
is an npm lifecycle script that runs automatically after `npm install`. So
every time someone installs dependencies, Husky re-registers the hooks
(it points git's `core.hooksPath` at `.husky/_`). New clones get the hook for
free after their first `npm install`.

**What's committed vs. ignored:**
- `.husky/pre-push` — **committed** (the hook you wrote).
- `.husky/_/` — **gitignored** (Husky's internal runner; it regenerates).

**The hook itself** (`set -e` makes it abort on the first failing check):

```sh
set -e
npm run lint:ci
npm run format:check
npm run typecheck
npm test
```

**The escape hatch:** `git push --no-verify` skips the hook. Use it only for
genuine emergencies (e.g., pushing a docs-only hotfix while a flaky local
environment blocks tests). It is **not** a way to dodge a real failure — CI
will still catch it, and you'll have shipped a known-bad commit to `main`.

---

## Recipes

**Run the gates locally (before pushing):**
```bash
# frontend or backend:
npm run lint:ci && npm run format:check && npm run typecheck && npm test
```

**Fix a formatting failure:** run the mutating twin, then re-check:
```bash
npm run format      # rewrites files
npm run format:check
```

**Fix a lint failure:** `npm run lint` (auto-fixes what it can), then read and
fix whatever remains by hand.

**A pre-push hook is blocking a legitimate push:** first ask *why* it's
failing — that's the hook doing its job. Fix the underlying issue. Only reach
for `--no-verify` in a real emergency.

**CI is red on `main`:** open the repo's **Actions** tab, click the failed
run, read the failing step's log. Reproduce locally with the same command
(e.g. `npm run typecheck`), fix, push again. Because we commit straight to
`main`, treat a red `main` as "drop what you're doing and get it green."

**Add a new gate:** add an npm script, add a step to the relevant `ci.yml`
(cheapest-first ordering), and — if it should also run locally — add a line to
`.husky/pre-push`. Update the "What's enforced where" table in this doc.

---

## Future / not yet

These are **not built today** — listed so you know where they'd go.

- **CD (deployment).** There is no deploy pipeline. The Docker image build was
  removed (the backend runs on the host in dev; see the backend `README`).
  When deployment is added, it becomes a **separate job** that runs *after* the
  build/test gates pass, reads real secrets from **GitHub Secrets**, and
  deploys to a staging environment before production. A rollback path should
  land with it.
- **Tighten `npm audit` to blocking.** Today it's informational because the
  high/critical findings are **transitive, build-time** dependencies (e.g.
  `postcss` via Next's toolchain; `lodash`/`multer`/`webpack` via NestJS) with
  no clean forward fix. After a deliberate dependency-upgrade pass, flip it to
  a blocking gate (remove `continue-on-error`).
- **Dependabot / Renovate.** Automated weekly dependency-update PRs.
- **Branch protection.** Only relevant if the project moves off the
  commit-straight-to-`main` model to a PR-based flow; then CI status checks
  would be *required* before merge.

---

## Glossary

- **CI (Continuous Integration):** automatically checking every change
  (lint/types/tests/build) on a server so breakage is caught immediately.
- **CD (Continuous Delivery/Deployment):** automatically shipping a change that
  passed CI to an environment. *(Not set up yet.)*
- **Gate:** a check that can pass or fail. A **blocking** gate stops the
  push/run on failure; an **informational** one only reports.
- **Hook (git hook):** a script git runs automatically at a lifecycle point
  (we use **pre-push**).
- **Shift left:** catch problems as early/cheap as possible (lint before build,
  local hook before CI).
- **Transitive dependency:** a package you didn't install directly — it was
  pulled in by one of your dependencies. Most `npm audit` noise is here.
- **Service container:** a throwaway container (e.g. Postgres) GitHub starts
  alongside a CI job so tests have a real dependency to talk to.
- **`npm ci`:** a clean, lockfile-exact install used in CI (vs. `npm install`,
  which may update the lockfile).

---

## When to update THIS file

Update this guide whenever the CI/CD reality changes:

- A gate is added, removed, or moved between pre-push and CI → update the
  "What's enforced where" table and the relevant per-repo section.
- A workflow file or `.husky/pre-push` changes meaningfully → update the
  matching excerpt (remember: the file is the source of truth, this doc
  explains it).
- The standard stack changes (tool, Node version, package manager) → update
  "The standard stack".
- CD/deployment lands → move it out of "Future" into a real section, and
  document the secrets flow.

This file is registered in the root `CLAUDE.md` under "Where authoritative docs
live" so humans and agents can find it.

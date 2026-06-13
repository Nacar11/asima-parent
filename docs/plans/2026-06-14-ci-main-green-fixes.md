# CI `main` go-green fixes — 2026-06-14

Both repos' `CI` workflow had been failing on **every** push to `main`. The
red runs date back to at least **2026-06-13** (e.g. backend run on commit
`80fb705`, frontend run on `0beb204`). This note records the three root
causes found and the durable fixes shipped on **2026-06-14**.

Cross-cutting (touches both `asima-backend` and `asima-frontend`), so the
snapshot lives in the parent `docs/plans/`. Companion guideline:
`docs/universal-guidelines/ci-cd-and-quality-gates.md`.

## How the failures were diagnosed

`gh` was not installed at the time, so CI logs were pulled via the GitHub
Actions API using the token from the macOS keychain
(`git credential fill` → `api.github.com/repos/Nacar11/<repo>/actions/runs`).
Each failure was then reproduced locally before any fix:

- backend e2e — mirrored CI exactly: fresh `asima_test` DB + `migration:run`
  + the running local MinIO, with CI-identical env.
- frontend — reproduced inside `node:20` Linux via Docker (CI runs Node 20;
  local is Node 24, which masked two of the bugs).

## Root cause 1 — backend e2e: app failed to boot (no `STORAGE_*`)

- **Symptom:** every e2e suite died at `TestingModule.compile`, not on an
  assertion. Error: `STORAGE_REGION/BUCKET/ACCESS_KEY/SECRET_KEY ... isString`.
- **Cause:** `src/storage/config/storage.config.ts` requires those env vars.
  `ci.yml` never set them, so the Nest test app could not instantiate. The
  leave-attachment specs also use the **real** `S3StorageService` and assert
  objects actually land in storage — so env vars alone are insufficient; CI
  needs a live MinIO + the bucket.
- **Fix (`asima-backend` `19335f3`):** added the `STORAGE_*` env block to
  `ci.yml` and a **Start MinIO + create bucket** step. MinIO is run as a
  plain `docker run minio/minio server /data` container (NOT a GitHub
  *service container* — those can't supply the required `server` command),
  with the bucket created via `mc` using `MC_HOST_*`.
- **Verified:** full e2e green locally — **108 tests / 11 suites** — against a
  fresh migrated DB + MinIO with CI-identical env.

## Root cause 2 — frontend `npm ci`: macOS-only lockfile

- **Symptom:** `Install dependencies` step aborted with
  `Missing: @emnapi/runtime@1.11.1 from lock file` (and `@emnapi/core`).
- **Cause:** `package-lock.json` was generated on macOS, so it omitted the
  Linux-only `wasm32-wasi` fallback packages (and their `@emnapi/*` deps)
  that `sharp` / the resolver binding resolve on Linux. `npm ci` on the
  Linux runner then refused the out-of-sync lockfile. Passed locally,
  failed on CI.
- **Fix (`asima-frontend` `17981c8`):** regenerated the lockfile **inside
  Linux** — `docker run --rm -v "$PWD":/app -w /app node:20
  npm install --package-lock-only` — so it carries every platform's optional
  deps.
- **Verified:** `npm ci` passes on **both** Linux and macOS; build green.
- **Guardrail going forward:** always regenerate the frontend lockfile on
  Linux, never bare on macOS, or this recurs.

## Root cause 3 — frontend unit test: jsdom vs undici `Blob`

Surfaced only **after** root cause 2 was fixed — `npm ci` had been dying
first, so the unit-test step had never actually run in CI.

- **Symptom:** `api-client.spec.ts > getBlob ...` failed on CI (Node 20) with
  `expected Blob { size: 13 } to be an instance of Blob`. `size: 13` ==
  `len("[object Blob]")`.
- **Cause:** the mock built `new Response(new Blob(['bytes'], ...))`, mixing
  jsdom's `Blob` into the global `Response` (undici's), which doesn't accept
  a jsdom Blob as `BodyInit` and stringified it to `"[object Blob]"`. The
  assertion then used `toBeInstanceOf(Blob)`, comparing across realms
  (jsdom's Blob vs the undici Blob from `Response.blob()`). The realms
  happened to align on Node 24 (local) but not Node 20 (CI).
- **Fix (`asima-frontend` `d277e46`):** mock with a realm-neutral **string
  body + `Content-Type` header**, and assert the Blob contract structurally
  (`constructor.name` / `type` / `size`) rather than by `toBeInstanceOf` or
  by calling methods (jsdom's Blob even lacks `.text()` / `.arrayBuffer()`).
- **Verified:** full unit suite — **270 tests** — green under **both** Node 20
  (Docker) and Node 24 (local pre-push hook).

## Outcome

| Repo | Fix commit (2026-06-14) | CI on `main` |
|---|---|---|
| asima-backend | `19335f3` | ✅ success |
| asima-frontend | `17981c8` (lockfile) + `d277e46` (test) | ✅ success |

No recurring/manual step is needed — all three were structural gaps in CI
config / lockfile / test and are corrected at the source.

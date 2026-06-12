# MinIO object storage + leave attachments

**Date:** 2026-06-12
**Status:** Planned
**Repos:** `asima-backend` (primary), `asima-parent` (docker-compose lives in backend), `asima-frontend` (leave form + download).
**Owner surface:** new `src/storage/` module + leave-requests wiring.

## Goal

Stand up object storage as the platform's first-class file/image facility,
and use it for its first concrete consumer: a **mandatory single attachment**
on `sick` and `bereavement` leave requests. MinIO runs as a new Docker
container for local development; deployed test/staging and production use AWS
S3. The two are the same S3 API behind one adapter — the environment only
changes configuration, not code paths.

## Decisions (locked during brainstorming, 2026-06-12)

| Question | Decision |
|---|---|
| Attachment requirement | **Always mandatory** for `sick` + `bereavement`. Submit is 422 without exactly one file. Other leave types reject an attachment. |
| Allowed file types | **Images (JPEG/PNG/WebP) + PDF.** Images get versions; PDFs stored as-is. |
| Download access | **Owner + the request's L1/L2 approvers + LEAVE admin/view holders.** Streamed through the API, permission-checked per request. |
| Scope this pass | **Storage module + leave attachment only.** Profile photos and other consumers are future work on the same foundation. |
| `local` env | MinIO container. |
| `deployed test/staging` + `prod` env | AWS S3. |
| Automated tests (CI Jest unit/e2e) | Storage **port is mocked / local MinIO** — never reach real AWS. |
| Upload flow | **One-step multipart submit** (file rides with the leave submit request). No orphan files. |
| Download mechanism | **Stream through the API** with a per-request permission check. No presigned URLs handed to clients (v0). |

## Core insight — one S3 client, not two code paths

MinIO and AWS S3 speak the same API. A single `S3Storage` adapter built on
`@aws-sdk/client-s3` serves both: pointed at the MinIO endpoint with
`forcePathStyle: true` locally, and at the AWS default endpoint (endpoint
unset, real region + creds) in deployed environments. The adapter sits behind
a `BaseStorageService` port so unit tests mock it and a future non-S3 driver
stays possible. **There is no `if (env === 'local')` branch in code** — the
selection is entirely in `STORAGE_*` config.

## Architecture

### New module `src/storage/` (hexagonal)

```
src/storage/
├── domain/
│   ├── stored-object.ts          # { bucket, key, content_type, size_bytes }
│   ├── upload-input.ts           # { key, body: Buffer, content_type }
│   └── file-version.ts           # VERSION enum: original | preview | thumbnail
├── config/
│   ├── storage.config.ts         # registerAs('storage') + env validator
│   └── storage-config.type.ts
├── base-storage.service.ts       # PORT: put / getStream / delete / exists
├── s3-storage.service.ts         # @aws-sdk/client-s3 impl (MinIO + AWS)
├── image-processor.service.ts    # sharp → { original, preview, thumbnail } buffers
├── file-validation.ts            # magic-byte sniff + size guard (pure)
├── attachment.service.ts         # orchestrator: validate→process→upload→persist row
└── storage.module.ts             # binds Base→S3, exports the port + AttachmentService
```

- **`file-validation`** inspects real magic bytes (not the client-sent MIME) for
  JPEG / PNG / WebP / PDF; rejects anything else with 422. Enforces
  `STORAGE_MAX_FILE_MB` (default 10).
- **`image-processor`** (sharp): `original` (as-uploaded bytes, unmodified),
  `preview` (≤1600px long edge, WebP, quality-optimized), `thumbnail`
  (≤256px long edge, WebP). PDFs skip this — `original` only. Sharp is
  configured with `limitInputPixels` to cap decoded dimensions, so a crafted
  "image bomb" can't exhaust memory even within the byte-size cap.
- **`s3-storage`** uses `PutObjectCommand` / `GetObjectCommand` /
  `DeleteObjectCommand` / `HeadObjectCommand`. `getStream` returns the body as
  a stream for the download endpoint to pipe.
- **`BaseStorageService`** is the abstract port `AttachmentService` depends on.
- **`AttachmentService`** is the orchestrator that owns "validate → process →
  upload → persist `attachments` row → return id (+ a rollback handle)" so the
  leave service stays focused on leave rules and every *future* consumer
  (profile photos, etc.) reuses the same seam (review fix — keeps the storage
  responsibility out of `LeaveRequestsService`). It depends on the storage
  port, the image processor, the validator, and the attachments repository.

### Bucket + key layout

One bucket (`STORAGE_BUCKET`, default `asima`). One key prefix per attachment:

```
leave-attachments/{attachment_id}/original.{ext}
leave-attachments/{attachment_id}/preview.webp      # images only
leave-attachments/{attachment_id}/thumbnail.webp    # images only
```

The `attachment_id` (DB row id) is the natural, collision-free prefix.

### Data model — generic `attachments` table

Reusable so future consumers (profile photos, etc.) share it; **not**
leave-specific.

`attachments`:
- `id` (PK)
- `bucket` (text)
- `object_key_prefix` (text, e.g. `leave-attachments/<uuid>` — a generated
  UUID, NOT the DB id; see Upload flow for why)
- `original_filename` (text)
- `content_type` (text)
- `size_bytes` (int)
- `kind` (enum `image | pdf`)
- `has_versions` (bool — true for images, false for PDFs)
- `owner_id` (FK users — who uploaded; backs the download access check)
- audit: `created_by`, `updated_by`, `deleted_by` (FK users), `created_at`,
  `updated_at`, `deleted_at` (soft delete)

`leave_requests` gains nullable `attachment_id` (FK `attachments`).

**Migrations** (per `database-migration-conventions.md`):
- New `CreateAttachmentsTable` migration, timestamped to run **before** the
  leave-requests table dependency is needed.
- The `attachment_id` column **folds into the existing
  `CreateLeaveRequestsTable` CREATE migration** — that table is still
  unreleased (dev-only), so no separate `Alter…` file. Order the attachments
  CREATE migration before the leave-requests CREATE migration so the FK
  resolves.

## Data flow

### Upload — one-step multipart submit

`POST /api/v1/users/me/leave-requests` becomes `multipart/form-data`: the
existing JSON fields as form fields + a single `file` part.
`me-leave-requests.controller` uses `FileInterceptor('file')`
(`@nestjs/platform-express`, memory storage — buffers the whole file in RAM,
bounded by the size cap). Service sequence:

1. **Enforce the size cap FIRST**, before any processing — reject oversize at
   the validation boundary so sharp never runs on an out-of-bound buffer.
   Then **validate type** (magic bytes, not the client MIME) → 422 on bad
   type/oversize.
2. **Enforce the rule**:
   - `leave_type ∈ {sick, bereavement}` and **no file** → 422
     (`attachment` required).
   - `leave_type ∉ {sick, bereavement}` and file present → 422 (attachment not
     accepted for this type — keeps payloads honest).
3. **Generate versions** (sharp) for images; PDFs pass through.
4. **Generate a UUID prefix** and **upload every version object to storage**
   under `leave-attachments/<uuid>/…` — all of this **before** opening any DB
   transaction.
5. **Persist in one short transaction**: insert the `attachments` row (storing
   the UUID `object_key_prefix`) and the `leave_request` referencing the new
   `attachment_id`, then commit.
6. **Compensate on failure**: storage uploads can't enlist in the SQL
   transaction. If the transaction throws after objects were written,
   best-effort `delete` the uploaded objects (logged on failure — orphan sweep
   is out of scope).

> **Why a UUID prefix, not the DB id (review fix):** keying objects by the
> DB-generated `attachment_id` would force the S3 uploads to happen *between*
> two SQL statements with the transaction open — pinning a pooled connection
> and holding locks across network I/O. Using a UUID lets all uploads finish
> before a single short transaction inserts both rows. Decoupling network I/O
> from the transaction is the goal; the compensating delete still covers a
> failed commit.

### Download — streamed, permission-checked

`GET /api/v1/leave-requests/:id/attachment?version=original|preview|thumbnail`
(default `original`):

1. Load the leave request + its attachment. 404 if either missing.
2. **Authorize against the request's SNAPSHOTTED approver columns** (review
   fix): the `leave_requests` row already carries `l1_approver_id` and
   `l2_approver_id` (snapshotted at submit). So the check is a plain field
   comparison — caller is the **owner** (`leave_request.employee_id ===
   req.user.id`), OR `req.user.id === leave_request.l1_approver_id`, OR
   `req.user.id === leave_request.l2_approver_id`, OR holds **`LEAVE:View`**
   admin permission (`system_admin` bypasses). Else 403. **No `approval_chains`
   lookup** — the snapshot is the authority for who approved *this* request,
   and the chain may have changed since. `l2_approver_id` is nullable
   (single-step chains); a null never matches a caller id, so no special case.
3. Reject `preview`/`thumbnail` for a PDF attachment (no such versions) → 404.
4. `getStream` from storage; pipe with correct `Content-Type` and a
   **sanitized** `Content-Disposition`. `original_filename` is user-controlled,
   so encode it per RFC 5987 (`filename*=UTF-8''<pct-encoded>`) and strip
   CR/LF + quotes to avoid header injection — never interpolate it raw into the
   header.

## Docker

`docker-compose.yml` (in `asima-backend`):
- **`asima-minio`** — `minio/minio`, command `server /data --console-address
  ":9001"`, ports `9000` (API) + `9001` (console), volume `asima-miniodata`,
  env `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` from `.env`, healthcheck on
  `/minio/health/ready`.
- **`asima-minio-init`** — one-shot `minio/mc` that waits for MinIO, then
  `mc mb --ignore-existing` the bucket (and sets it private). Exits 0.
- **`asima-api`** — add `STORAGE_*` env, `depends_on: asima-minio
  (service_healthy)`. Local in-container endpoint is `http://asima-minio:9000`;
  host-mode dev (`npm run start:dev`) uses `http://localhost:9000`.
- `.env.example` documents every `STORAGE_*` key + `MINIO_ROOT_*`.

## Config keys (`registerAs('storage')`, validated like `app.config.ts`)

| Key | Local (MinIO) | Deployed (AWS) | Notes |
|---|---|---|---|
| `STORAGE_ENDPOINT` | `http://asima-minio:9000` | *(empty)* | empty = AWS default endpoint |
| `STORAGE_REGION` | `us-east-1` | real region | MinIO ignores but SDK requires |
| `STORAGE_BUCKET` | `asima` | real bucket | one bucket, prefixed keys |
| `STORAGE_ACCESS_KEY` | MinIO root user | IAM key | |
| `STORAGE_SECRET_KEY` | MinIO root pw | IAM secret | |
| `STORAGE_FORCE_PATH_STYLE` | `true` | `false` | MinIO needs path-style |
| `STORAGE_MAX_FILE_MB` | `10` | `10` | validation cap |

Wire the `storage` namespace into `AllConfigType`.

## Phases (dependency-ordered)

### Phase 0 — Foundation: deps, config, docker
- Add deps: `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner` (kept for
  future presigned use), `sharp`, and dev dep `@types/multer` (review fix —
  `@nestjs/platform-express` bundles multer's runtime but NOT the
  `Express.Multer.File` type that `FileInterceptor`'s file param needs).
- `storage.config.ts` + `storage-config.type.ts`; register in `AllConfigType`
  and `app.module` config load. Env validator with the keys above.
- `.env.example`: `STORAGE_*` + `MINIO_ROOT_*`.
- `docker-compose.yml`: `asima-minio`, `asima-minio-init`, api env +
  `depends_on`.
- **AC:** `docker compose up` brings MinIO healthy and creates the bucket;
  `npm run build` passes with the new config wired.

### Phase 1 — Storage module
- domain types, `BaseStorageService` port, `S3Storage` impl,
  `ImageProcessorService` (sharp, with `limitInputPixels` set), `file-validation`,
  `AttachmentService` (orchestrator), `storage.module`.
- **Unit tests:** `file-validation` (magic bytes for each type, oversize → 422,
  spoofed MIME rejected), `image-processor` (sharp on a fixture image yields 3
  buffers with expected dimensions; PDF buffer skipped; oversized-dimension
  image rejected by `limitInputPixels`).
- **AC:** unit tests green; module compiles; port is mockable.

### Phase 2 — Attachments persistence
- `CreateAttachmentsTable` migration (audit cols + soft delete; `object_key_prefix`
  stores the UUID). **Timestamp window (review fix):** must be `> users-table`
  and `< 1778400000000` (`CreateLeaveRequestsTable`) so the folded
  `attachment_id` FK resolves — e.g. `1778350000000`.
- Fold `attachment_id` FK into `CreateLeaveRequestsTable` migration (still
  unreleased/dev-only, so no separate `Alter…` file).
- `attachment` entity, domain class, mapper, repository, base repo,
  persistence module.
- **Unit tests:** mapper `toDomain`/`toPersistence` round-trip.
- **AC:** `npm run db:fresh` applies cleanly; mapper tests green.

### Phase 3 — Leave attachment wiring (upload)
- `SubmitLeaveRequestDto`: keep JSON fields; controller switches to
  `multipart/form-data` + `FileInterceptor('file')`.
- `LeaveRequestsService`: owns only the leave-side rule (mandatory for
  sick/bereavement, rejected otherwise) and delegates the
  validate→process→upload→persist work to `AttachmentService` (review fix — no
  storage orchestration in the leave service). Upload happens via UUID prefix
  **before** a single short transaction that inserts the attachment +
  leave_request rows; compensating object delete on rollback.
- `leave_request` domain/entity/mapper carry `attachment_id`.
- **Unit tests:** rule enforcement (sick w/o file → 422; vacation w/ file →
  422; sick w/ valid image → uploads 3 versions + links id), storage port
  mocked; DB-failure path triggers the compensating delete.
- **E2E:** submit sick leave without file → 422; with a valid image → 201,
  attachment row exists, 3 version objects exist in (local/mock) storage.
- **AC:** all green; submitting non-sick/bereavement leave is unchanged.

### Phase 4 — Download endpoint
- `GET /leave-requests/:id/attachment?version=` on the leave-requests
  controller (or a focused attachments controller in the module).
- Permission check against the request's snapshotted `l1/l2_approver_id`
  (owner / snapshotted approver / `LEAVE:View` / `system_admin`); stream pipe
  with content headers + RFC 5987-encoded filename; PDF `preview`/`thumbnail`
  → 404.
- **E2E:** owner 200 (original + preview + thumbnail for image), approver 200,
  unrelated employee 403, missing request 404, PDF preview 404.
- **AC:** all green.

### Phase 5 — Frontend (asima-frontend)
- Leave submit form: when `leave_type` is `sick` or `bereavement`, show a
  required single-file input (accept `image/jpeg,image/png,image/webp,
  application/pdf`); submit as `multipart/form-data`. Client-side guard mirrors
  the server (block submit without a file), but the server stays the boundary.
- Request detail / approvals view: an attachment thumbnail/preview (image) or
  PDF link that hits the download endpoint; falls back to a PDF icon.
- Follows existing leave-feature component + api patterns; Zod schema +
  api-client call for the download URL.
- **Unit tests:** form requires file for sick/bereavement; download link
  renders for a request with an attachment.
- **AC:** all green; lint + build pass.

## Testing strategy summary

- **Unit** (no network): validation, image processing, leave rule — storage
  port mocked.
- **E2E** (backend): runs against local MinIO (or a mocked storage provider in
  CI) — never real AWS. Covers upload rule, version creation, download
  permissions.
- New backend deps: `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner`,
  `sharp`, dev `@types/multer`. `@nestjs/platform-express` (which bundles
  multer's runtime) is already present; only the `@types/multer` types are
  missing.

## Risks / notes

- **sharp** ships native binaries — ensure the backend Docker image
  (`docker/Dockerfile`, Alpine) installs compatible sharp (`libvips`); pin a
  version known to work on the base image. Verify in Phase 0/1.
- **Storage ↔ DB atomicity** is eventual: compensating delete on rollback,
  with logging. A periodic orphan sweep is **out of scope** but noted as
  future hardening.
- **`test → AWS`** means *deployed* test/staging, not CI. CI must never carry
  real AWS creds; e2e uses local MinIO or a mocked provider.

## Out of scope (YAGNI)

Profile photos & other storage consumers; presigned public URLs; antivirus
scanning; multi-file attachments; PDF page-render thumbnails; orphan-object
sweeper; per-tenant buckets.

## Acceptance criteria (feature-level)

1. MinIO runs as a local Docker container with a created bucket; deployed envs
   use AWS S3 via the same adapter, switched only by `STORAGE_*` config.
2. Submitting `sick`/`bereavement` leave without exactly one valid file is
   rejected (422); with a valid image it stores `original` + `preview` +
   `thumbnail`; with a PDF it stores `original`.
3. Non-sick/bereavement leave rejects an attachment and otherwise behaves as
   before.
4. The attachment is downloadable only by the owner, the request's approvers,
   and LEAVE admins — streamed through the API, no presigned URLs.
5. All new code is unit + e2e tested; CI never touches real AWS.

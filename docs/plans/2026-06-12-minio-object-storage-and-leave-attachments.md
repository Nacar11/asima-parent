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
└── storage.module.ts             # binds Base→S3, exports the port + helpers
```

- **`file-validation`** inspects real magic bytes (not the client-sent MIME) for
  JPEG / PNG / WebP / PDF; rejects anything else with 422. Enforces
  `STORAGE_MAX_FILE_MB` (default 10).
- **`image-processor`** (sharp): `original` (as-uploaded bytes, unmodified),
  `preview` (≤1600px long edge, WebP, quality-optimized), `thumbnail`
  (≤256px long edge, WebP). PDFs skip this — `original` only.
- **`s3-storage`** uses `PutObjectCommand` / `GetObjectCommand` /
  `DeleteObjectCommand` / `HeadObjectCommand`. `getStream` returns the body as
  a stream for the download endpoint to pipe.
- **`BaseStorageService`** is the abstract port the leave service depends on.

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
- `object_key_prefix` (text, e.g. `leave-attachments/42`)
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
(`@nestjs/platform-express`, memory storage). Service sequence:

1. **Validate** the file (magic bytes + size) → 422 on bad type/oversize.
2. **Enforce the rule**:
   - `leave_type ∈ {sick, bereavement}` and **no file** → 422
     (`attachment` required).
   - `leave_type ∉ {sick, bereavement}` and file present → 422 (attachment not
     accepted for this type — keeps payloads honest).
3. **Generate versions** (sharp) for images; PDFs pass through.
4. **Upload** every version object to storage under the attachment prefix.
   The `attachment_id` is needed for the key, so allocate it first (insert the
   `attachments` row inside the transaction, then upload using its id, OR use a
   generated UUID prefix; plan uses: insert attachments row → get id → upload).
5. **Persist transactionally**: within one DB transaction, insert the
   `attachments` row and the `leave_request` referencing `attachment_id`.
6. **Compensate on failure**: storage uploads can't enlist in the SQL
   transaction. If the DB step throws after objects were written, best-effort
   `delete` the uploaded objects (logged on failure — orphan sweep is out of
   scope).

> Ordering note: because the object key uses `attachment_id`, the attachments
> row is inserted first (to get the id), objects are uploaded, then the
> leave_request insert completes the transaction. If the leave_request insert
> fails, the transaction rolls back the attachments row and the compensating
> delete removes the objects.

### Download — streamed, permission-checked

`GET /api/v1/leave-requests/:id/attachment?version=original|preview|thumbnail`
(default `original`):

1. Load the leave request + its attachment. 404 if either missing.
2. **Authorize**: caller is the **owner** (`leave_request.employee_id ===
   req.user.id`), OR an **L1/L2 approver** on this request's chain, OR holds
   **`LEAVE:View`** admin permission (`system_admin` bypasses). Else 403.
3. Reject `preview`/`thumbnail` for a PDF attachment (no such versions) → 404.
4. `getStream` from storage; pipe with correct `Content-Type` and
   `Content-Disposition: inline; filename="<original_filename>"`.

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
  future presigned use), `sharp`.
- `storage.config.ts` + `storage-config.type.ts`; register in `AllConfigType`
  and `app.module` config load. Env validator with the keys above.
- `.env.example`: `STORAGE_*` + `MINIO_ROOT_*`.
- `docker-compose.yml`: `asima-minio`, `asima-minio-init`, api env +
  `depends_on`.
- **AC:** `docker compose up` brings MinIO healthy and creates the bucket;
  `npm run build` passes with the new config wired.

### Phase 1 — Storage module
- domain types, `BaseStorageService` port, `S3Storage` impl,
  `ImageProcessorService` (sharp), `file-validation`, `storage.module`.
- **Unit tests:** `file-validation` (magic bytes for each type, oversize → 422,
  spoofed MIME rejected), `image-processor` (sharp on a fixture image yields 3
  buffers with expected dimensions; PDF buffer skipped).
- **AC:** unit tests green; module compiles; port is mockable.

### Phase 2 — Attachments persistence
- `CreateAttachmentsTable` migration (audit cols + soft delete).
- Fold `attachment_id` FK into `CreateLeaveRequestsTable` migration; order
  attachments-create before leave-create.
- `attachment` entity, domain class, mapper, repository, base repo,
  persistence module.
- **Unit tests:** mapper `toDomain`/`toPersistence` round-trip.
- **AC:** `npm run db:fresh` applies cleanly; mapper tests green.

### Phase 3 — Leave attachment wiring (upload)
- `SubmitLeaveRequestDto`: keep JSON fields; controller switches to
  `multipart/form-data` + `FileInterceptor('file')`.
- `LeaveRequestsService`: validation rule (mandatory for sick/bereavement,
  rejected otherwise), version generation, storage upload, transactional
  insert of attachment + leave_request, compensating object delete on rollback.
- `leave_request` domain/entity/mapper carry `attachment_id`.
- **Unit tests:** rule enforcement (sick w/o file → 422; vacation w/ file →
  422; sick w/ valid image → uploads 3 versions + links id), storage port
  mocked; DB-failure path triggers compensating delete.
- **E2E:** submit sick leave without file → 422; with a valid image → 201,
  attachment row exists, 3 version objects exist in (local/mock) storage.
- **AC:** all green; submitting non-sick/bereavement leave is unchanged.

### Phase 4 — Download endpoint
- `GET /leave-requests/:id/attachment?version=` on the leave-requests
  controller (or a focused attachments controller in the module).
- Permission check (owner / approver / `LEAVE:View` / `system_admin`); stream
  pipe with content headers; PDF `preview`/`thumbnail` → 404.
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
  `sharp`. NestJS Express (`@nestjs/platform-express`, multer) already present.

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

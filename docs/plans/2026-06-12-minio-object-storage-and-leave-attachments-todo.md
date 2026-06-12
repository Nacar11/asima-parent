# MinIO object storage + leave attachments — TODO

Plan: `2026-06-12-minio-object-storage-and-leave-attachments.md`

## Phase 0 — Foundation: deps, config, docker
- [ ] Add backend deps: `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner`, `sharp`
- [ ] Verify `sharp` builds on the Alpine backend image (`docker/Dockerfile`, libvips) — pin a working version
- [ ] `src/storage/config/storage.config.ts` + `storage-config.type.ts` (registerAs 'storage') with env validator
- [ ] Register `storage` in `AllConfigType` + `app.module` config load
- [ ] `.env.example`: `STORAGE_*` + `MINIO_ROOT_*` keys documented
- [ ] `docker-compose.yml`: `asima-minio` service (volume, healthcheck)
- [ ] `docker-compose.yml`: `asima-minio-init` one-shot (mc) creates private bucket
- [ ] `docker-compose.yml`: `asima-api` gains `STORAGE_*` env + `depends_on: asima-minio (healthy)`
- [ ] AC: `docker compose up` → MinIO healthy + bucket created; `npm run build` passes

## Phase 1 — Storage module
- [ ] domain: `stored-object.ts`, `upload-input.ts`, `file-version.ts`
- [ ] `base-storage.service.ts` port (put / getStream / delete / exists)
- [ ] `s3-storage.service.ts` (@aws-sdk/client-s3; forcePathStyle from config)
- [ ] `image-processor.service.ts` (sharp → original/preview/thumbnail buffers)
- [ ] `file-validation.ts` (magic-byte sniff + size guard, pure)
- [ ] `storage.module.ts` (bind Base→S3, export port + helpers)
- [ ] Unit: file-validation (each type, spoofed MIME, oversize → 422)
- [ ] Unit: image-processor (3 buffers + dims on fixture; PDF skipped)
- [ ] AC: unit green; module compiles; port mockable

## Phase 2 — Attachments persistence
- [ ] `CreateAttachmentsTable` migration (cols + audit + soft delete)
- [ ] Fold `attachment_id` FK into `CreateLeaveRequestsTable` migration; order attachments-create first
- [ ] `attachment` entity, domain class, mapper, repository, base repo, persistence module
- [ ] Unit: mapper round-trip
- [ ] AC: `npm run db:fresh` clean; mapper tests green

## Phase 3 — Leave attachment wiring (upload)
- [ ] `me-leave-requests.controller`: `multipart/form-data` + `FileInterceptor('file')`
- [ ] Service: mandatory-for-sick/bereavement rule; reject attachment on other types (422)
- [ ] Service: version generation → storage upload under `leave-attachments/{id}`
- [ ] Service: transactional insert (attachment + leave_request) + compensating object delete on rollback
- [ ] `leave_request` domain/entity/mapper carry `attachment_id`
- [ ] Unit: rule enforcement + happy path + DB-failure compensating delete (storage mocked)
- [ ] E2E: sick w/o file → 422; sick w/ image → 201 + 3 versions; vacation w/ file → 422
- [ ] AC: green; non-sick/bereavement submit unchanged

## Phase 4 — Download endpoint
- [ ] `GET /leave-requests/:id/attachment?version=` (default original)
- [ ] Permission check: owner / approver / `LEAVE:View` / `system_admin`
- [ ] Stream pipe with `Content-Type` + `Content-Disposition`; PDF preview/thumbnail → 404
- [ ] E2E: owner 200 (3 versions), approver 200, stranger 403, missing 404, PDF preview 404
- [ ] AC: green

## Phase 5 — Frontend (asima-frontend)
- [ ] Leave submit form: required single-file input for sick/bereavement (accept images + pdf); multipart submit
- [ ] Client-side guard mirrors server (block submit without file)
- [ ] Request detail / approvals: image preview or PDF link via download endpoint; PDF-icon fallback
- [ ] Zod schema + api-client call for the attachment download
- [ ] Unit: form requires file for sick/bereavement; download link renders
- [ ] AC: green; lint + build pass

## Done = all five feature-level acceptance criteria in the plan met.

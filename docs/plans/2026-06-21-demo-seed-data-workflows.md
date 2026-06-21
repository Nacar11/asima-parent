# Demo seed data for the workflow tables

**Date:** 2026-06-21
**Scope:** asima-backend (seed services + data only — no schema, no API changes)
**Status:** planned, five-axis reviewed (C1–C3 + I1–I5 folded in); not started

## Why this, why now

The workflow-heavy modules — **approval chains, leave requests, the approvals
inbox, time corrections** — have **no seeders**, so their tables are empty after
`npm run seed`. The corresponding pages render nothing, which makes the app hard
to explore and demo. Everything else (users, schedules, time entries,
compensation) is already seeded.

This is independent of the OT foundations (B holiday / C multipliers / D
overnight / E claim) — those are *new tables that don't exist yet* and can't be
seeded. This plan only enriches seed data for **tables that already exist**.

## What's empty today

| Seeded ✅ | Empty after seed ❌ |
|---|---|
| permissions, roles, users (+ leave allocations) | **approval_chains** |
| time_entries, work_schedules | **leave_requests** |
| employee_compensations | **time_correction_requests** |
| | `attachments` (only the placeholder, below) |

## Decisions locked during brainstorming

1. **Repo-based seeders, one per table** (the existing pattern: each seed
   service inserts via its TypeORM repository, idempotent by natural key).
   Rejected service-driven seeding (needs JWT actors, throttling, harder
   idempotency). The one exception is the per-employee placeholder attachments
   (below), which go through the attachment service so `sharp`/MinIO produce
   valid renditions.
2. **Attachment-backed leave IS seeded** (reversing the earlier "skip" lean),
   using a **placeholder image**. Attachments are **owner-scoped**
   (`uploadForOwner` stamps `owner_id = employee_id`), so the placeholder is
   created **per employee** that has a sick/bereavement row — not one global row
   (review R1).
3. **No schema or API changes.** Only new seed services + a `compensation.json`
   tweak + a gitignored placeholder asset.

## Review findings folded in (2026-06-21, agent-skills:code-review)

- **C1 — balance-consuming leave must use allocation-backed types.** Only
  `vacation` + `sick` have allocations (`leave-allocations.constants.ts:26-28`);
  balance is derived (`leave-balance.service.ts:8-13`), so approving a
  zero-grant type (`birthday`/`emergency`/`bereavement`) drives its balance
  negative — a state submit would reject. **Resolution:** balance-consuming
  states (`pending_l1`, `pending_l2`, `approved`) use **vacation or sick** only,
  with per-employee consuming days kept **≤ the 10-day allocation** (small 1–2
  day ranges). Zero-grant types appear **only in non-consuming states**
  (`rejected`, `cancelled`). `sick`/`bereavement` rows carry the placeholder
  attachment (see below).
- **C2 — `pending_l2` requires a 2-level chain.** It is meaningless without an
  L2 approver and is indexed on `l2_approver_id WHERE status='pending_l2'`
  (`1778400000000-CreateLeaveRequestsTable.ts:80-82`). **Resolution:** only
  employees whose seeded chain has a **step 2** get `pending_l2` rows.
- **C3 — self-approval is a DB CHECK** `approver_id <> employee_id`
  (`1778300000000-CreateApprovalChainsTable.ts:40`). **Resolution:** the org
  mapping (below) routes every employee to a **different** person, and the
  seeder asserts `approver_id !== employee_id` before insert.
- **I1 — single source of truth for approvers.** `l1/l2_approver_id` is
  snapshotted from the active chain (`1778400000000:7`). The request seeders
  **read the seeded `approval_chains`** (loaded once into a map) rather than
  re-deriving the mapping — no drift.
- **I2 — `working_days` must be weekday-consistent and ≥ 0.5**
  (`1778400000000:60-63`); no holiday calendar exists yet. **Resolution:** seed
  **single weekday** ranges (`working_days = 1`), never weekends; half-day only
  on a single date.
- **I3 — generate rows from a compact spec, not raw JSON.** Each leave/correction
  row has ~10 interdependent fields. The seed service generates rows in code
  from a per-employee spec (`{ type, state, day_offset, portion? }`) and fills
  the derived fields once, so an inconsistent row is unrepresentable.
- **I4 — corrections use the real columns** (R2/R8): `target_entry_id`
  (**nullable** — null for a "missing punch"), `work_date`, `proposed_time_in`
  (NOT NULL), `proposed_time_out` (nullable; `> proposed_time_in` when present),
  `reason` (NOT NULL). Seed a mix — some rows reference a seeded `time_entry`,
  some leave `target_entry_id` null.
- **I5 — status↔decision invariant, enforced centrally:** `pending_*` ⇒ all
  decision/cancel fields null; `approved`/`rejected` ⇒ `decided_at` +
  `decided_by` + `decision_path` set; `cancelled` ⇒ `cancelled_at` +
  `cancelled_by` set.
- **S3 (resolved during review):** `OPERATIONS_MANAGER` (Mary) has **no** approve
  permission — only `PROJECT_MANAGER`/`TECHNICAL_DIRECTOR` hold `LEAVE:Approve` /
  `TIME_CORRECTION:Approve`, and `HR_ADMIN` holds the `ApproveAny` override. The
  org model therefore routes **only to PM / TD / HR**; Mary is an approvee only.

### Second review pass (2026-06-21, R1–R9)

- **R1 — attachments are owner-scoped**, so the placeholder is **per employee**
  (`owner_id = employee_id`), not one shared row (folded into decision #2 + the
  placeholder section + Task 2). Verified the inbox query surfaces seeded pending
  rows correctly (`approvals.service.ts:62-79,135`) — the seed-to-surface
  approach is sound.
- **R2 — correct time-correction columns:** `target_entry_id` (nullable),
  `proposed_time_in` (NOT NULL), `proposed_time_out` (nullable), `reason`
  (NOT NULL) — not "time_entry_id / requested in-out" (folded into Task 3 + I4).
- **R3 — the attachment path needs `StorageModule` wired into `SeedModule`**
  (it currently imports only Config + TypeOrm). Added as an explicit Task 2
  sub-step, with a lighter `storage.put` + repo-insert fallback noted.
- **R4 — seeder attachment+leave is non-atomic** (the app does both in one txn);
  the idempotency check (skip when the leave row exists) is the guard against
  re-uploading / orphans.
- **R7 — `reason` is NOT NULL on corrections;** every seeded correction sets one.
- **R8 — `target_entry_id` is nullable:** seed a mix — some rows reference a
  seeded entry, some are null ("missing punch").
- **R9 — confirm the `approval_chains` active-row predicate** (entity
  `started_at`/`ended_at` vs the migration's `effective_at` uniqueness) before
  writing Task 1's idempotency check.

## The approval-chain org model

Pools (from the user seed): **PM** = James Wilson, Robert Johnson, Linda Miller ·
**TD** = Karen Taylor, Michael Jones, Fred Bloggs, Joe Bloggs, Alice Bob · **HR**
= Jane Smith, John Doe. `admin` (SUPER_ADMIN) is skipped (not an employee).

| Employee's role | L1 approver | L2 approver | Levels |
|---|---|---|---|
| EMPLOYEE | `PM[i % 3]` | `TD[i % 5]` | 2 |
| PROJECT_MANAGER | `TD[i % 5]` | Jane Smith (HR) | 2 |
| TECHNICAL_DIRECTOR | Jane Smith (HR) | — | 1 |
| OPERATIONS_MANAGER (Mary) | Jane Smith (HR) | — | 1 |
| HR_ADMIN | the *other* HR (Jane↔John) | — | 1 |

Round-robin (`i` = index within role) spreads inboxes across **every PM and TD**
so each approver has something to act on. All chosen approvers can act (PM/TD via
chain, HR via `ApproveAny`); none equals the employee. Idempotent by
`(employee_id, step)` among active rows (matches the partial unique index).

## Data shape (the spread)

Per employee, generated from a compact spec so **every page has content**:

- **Leave** (each employee gets ~4 rows): one `pending_l1` (vacation, so it sits
  in the L1 approver's inbox), one `pending_l2` *only if the chain has step 2*
  (vacation/sick), one `approved` (vacation or sick, decided by the chain), one
  `rejected` **or** `cancelled` (may use birthday/emergency/bereavement — no
  balance impact). Consuming days per employee ≤ allocation.
- **Time corrections** (~2 rows): one `pending_l1` and one `approved`/`rejected`,
  each targeting a seeded `time_entry` work_date with a valid requested window.
- Attachment-required types (`sick`, `bereavement`) reference a per-employee
  placeholder attachment (R1).

## The placeholder attachment

A mock image stands in for medical-cert / death-notice uploads:

- **Asset:** `asima-parent/public/seed-attachment-placeholder.png` (800×560 PNG,
  generated with `sharp`). `public/` is **gitignored** in `asima-parent` — the
  file is local/dev-only and regenerated from the snippet in this plan.
- **Wiring (R3):** `uploadForOwner` lives in `AttachmentService`, which
  `SeedModule` does **not** currently provide (it imports only `ConfigModule` +
  `TypeOrmModule`). Task 2 must import `StorageModule` into `SeedModule` and
  confirm `STORAGE_*` loads in the seed `ConfigModule`. **Fallback** if that's
  heavy: a direct `storage.put` + `attachments` repo insert (then the seeder
  must set `object_key_prefix`/`kind` itself and skip WebP versions).
- **Seeding (R1):** for each employee with a seeded `sick`/`bereavement` row,
  hand the placeholder buffer to `uploadForOwner({ file, owner_id: <employee>,
  actor_id: <admin> })` so `original` + WebP versions land in MinIO and the
  `attachments` row is valid; reference that **per-employee** `attachment_id`.
  Idempotent: skip the whole row when the employee's leave row already exists.
- **Non-atomic (R4):** unlike the app (attachment + leave row in one txn,
  `leave-requests.service.ts:162`), the seeder uploads then inserts separately,
  so a mid-run crash can orphan an attachment/object. Harmless for a dev seeder;
  the "skip when the leave row exists" check prevents re-uploading on re-run.
- **Best-effort:** if the placeholder file is absent (fresh clone, CI) **or**
  MinIO is unreachable, the seeder **skips the sick/bereavement rows with a log**
  (never seeding one with a null `attachment_id`) and seeds everything else —
  `npm run seed` never breaks.

Regenerate the placeholder:
```bash
node -e "const s=require('./asima-backend/node_modules/sharp');\
s(Buffer.from('<svg xmlns=\"http://www.w3.org/2000/svg\" width=\"800\" height=\"560\"><rect width=\"800\" height=\"560\" fill=\"#eef2f7\"/><text x=\"400\" y=\"280\" font-size=\"44\" font-weight=\"bold\" fill=\"#334155\" text-anchor=\"middle\" font-family=\"sans-serif\">PLACEHOLDER</text></svg>'))\
.png().toFile('public/seed-attachment-placeholder.png')"
```

## Task list

### Task 1: Approval-chain seeder
**Description:** `ApprovalChainSeedService` → `approval_chains`. Build the role
pools, apply the org-model mapping (round-robin), insert step-1 (+ step-2 where
applicable) active rows. Assert `approver_id !== employee_id`. Wire into
`seed.module.ts` + `run-seed.ts` **after** `UserSeedService`.
**Acceptance criteria:**
- [ ] Every non-admin employee has ≥1 active chain row; PMs/EMPLOYEEs have a step 2.
- [ ] No row has `approver_id == employee_id`; all approvers are PM/TD/HR.
- [ ] `npm run seed` twice → no duplicate chains (idempotent by `(employee_id, step)`).
**Prereq (R9):** confirm the active-row predicate before coding idempotency — the
entity exposes `started_at`/`ended_at` but the migration's uniqueness uses
`effective_at`; match whichever the partial unique index actually keys on.
**Dependencies:** users. **Files:** `seeds/approval-chain/*`, `run-seed.ts`,
`seed.module.ts`. **Scope:** M.

### Task 2: Leave-request seeder (+ placeholder attachment)
**Description:** `LeaveRequestSeedService` → `leave_requests`. Load the seeded
chains into a map (I1); for each employee generate rows from the compact spec
(I3), filling approver + decision fields per the status↔decision invariant (I5);
single weekday ranges (I2); consuming states use vacation/sick within allocation
(C1); `pending_l2` only with a step-2 chain (C2). For each employee with a
sick/bereavement row, create a **per-employee** placeholder attachment (R1),
best-effort.
**Sub-step (R3):** wire `StorageModule` into `SeedModule` (confirm `STORAGE_*`
loads in the seed `ConfigModule`) so the seeder can call `AttachmentService.
uploadForOwner`; or take the `storage.put` + repo-insert fallback.
**Acceptance criteria:**
- [ ] Approver inboxes (PM/TD/HR) show pending rows; approved/rejected/cancelled
  populate history; balances stay non-negative.
- [ ] No `pending_l2` row for a single-level employee; no zero-grant type in a
  consuming state.
- [ ] Each sick/bereavement row references a **per-employee** placeholder
  attachment (`owner_id = employee`); the whole row is skipped-with-log when the
  asset/MinIO is missing (never seeded with a null `attachment_id`).
- [ ] Idempotent by `(employee_id, leave_type, start_date)`; re-run does not
  re-upload (skip when the leave row exists).
**Dependencies:** Task 1, leave allocations, storage/MinIO (optional). **Files:**
`seeds/leave-request/*`, `run-seed.ts`, `seed.module.ts`. **Scope:** L.

### Task 3: Time-correction seeder
**Description:** `TimeCorrectionRequestSeedService` → `time_correction_requests`.
Same chain-sourced approver + invariant logic. Columns (R2): `work_date`,
`proposed_time_in` (NOT NULL), `proposed_time_out` (nullable), `reason`
(NOT NULL, always set), `target_entry_id` (nullable — mix of seeded-entry and
null "missing punch", R8).
**Acceptance criteria:**
- [ ] Correction inboxes show pending rows; resolved ones populate history.
- [ ] `proposed_time_out > proposed_time_in` when present; `reason` set on every
  row; `target_entry_id` references a seeded entry or is null.
- [ ] Idempotent by `(employee_id, work_date)`.
**Dependencies:** Task 1, time entries. **Files:**
`seeds/time-correction/*`, `run-seed.ts`, `seed.module.ts`. **Scope:** M.

### Task 4: Compensation enrichment
**Description:** Extend `seeds/data/compensation.json` — a few more multi-row
histories + one more override — so the audit/history UIs show fuller data. No new
module.
**Acceptance criteria:** [ ] More employees have ≥2 comp rows; re-run idempotent.
**Dependencies:** none. **Files:** `seeds/data/compensation.json`. **Scope:** S.

### Checkpoint
- [ ] `npm run seed` twice = no-op; `npm run build`/`lint`/`test` green.
- [ ] Seed sanity spec (idempotency + every request's `l1_approver_id` resolves
  and ≠ employee), mirroring `seeds-grant-matrix.spec.ts` (review S2).
- [ ] Log in as a PM / TD / HR and confirm non-empty leave + correction inboxes.

## Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Hand-built rows violate a CHECK / balance goes negative | Med | Generate-from-spec (I3) + invariant helpers (C1/I2/I5); seed sanity spec |
| Placeholder/MinIO missing in CI breaks seed | Med | Attachment seeding is best-effort, skip-with-log (best-effort clause) |
| Approver routed to someone who can't act | Med | Org model routes only to PM/TD/HR (S3); assert in seeder |
| Cross-owner attachment fails view/download authz | Med | Placeholder is **per employee** (`owner_id = employee`), not shared (R1) |
| Orphan attachment on a mid-run crash (non-atomic, R4) | Low | Skip when the leave row exists → no re-upload; harmless dev artifact |
| StorageModule won't wire cleanly into SeedModule (R3) | Med | Confirm `STORAGE_*` in seed config; fallback = `storage.put` + repo insert |

## Out of scope
- *Unique real documents* per row — the per-employee placeholder is reused across
  that employee's attachment rows.
- Any schema/API change; the OT foundations (B/C/D/E); committing the binary asset.

## Phase 2 — Varied pending leave (added 2026-06-21)

**Why:** the seeded approvals inbox showed only identical `vacation, 2025-03-03`
rows — pending_l1/l2 were all vacation on the same date. This enriches the
**pending** leave so the inbox shows varied types, dates, and durations,
including a pending `sick` with the placeholder cert for an approver to review.

**Change (leave seeder + `buildLeaveRows` only — no schema/API/UI):** driven by
each employee's index:
- **Type rotation** on pending L1: `vacation → sick(+cert) → birthday → emergency`
  (`index % 4`); `sick` falls back to vacation when the employee has no attachment.
- **Staggered dates**: each employee's leave block starts on a different weekday
  (no longer all `2025-03-03`).
- **Duration variety**: a mix of single full days, **half-days**
  (vacation/sick/emergency only — `birthday` is whole-day by policy), and a few
  **2-day ranges**.
- Pending L2 stays vacation at a distinct date (bounded complexity).

**Half-day window:** derived from the employee's seeded `work_schedule`
(`expected_in`..`break_start` = first half; `break_end`..`expected_out` = second).
The service sets `start_time`/`end_time`; it downgrades a half-day to full when no
schedule exists. (The app's `assertSubmittableRange` rejects past dates, so the
seeder hand-computes the window — consistent with how it already hand-builds
past-dated rows.)

**Accepted trade-offs:**
- **Reverses C1 for pending rows:** zero-grant types (`birthday`/`emergency`) in a
  pending state reserve against a 0-day allocation → those types show a **negative
  balance** on the employee's leave page. Accepted for demo realism (was option 4).
- **Requires `npm run db:fresh`:** the date/type scheme changes and the seeder is
  idempotent *per natural key*, so re-running `seed` on the existing DB would add
  the new rows *alongside* the old all-vacation ones. `db:fresh` gives the clean set.

**Tests:** extend `buildLeaveRows` unit tests (type rotation, sick→vacation
fallback, half-day + 2-day durations, staggered dates, requires_attachment on
sick); re-seed via `db:fresh` and verify inbox variety via SQL.

# Implementation Plan: Leave Balances + Leave Page Revamp

> Cross-cutting feature (backend + frontend). Audit snapshot, 2026-05-31.
> This `docs/plans/` file is the **only committed** plan artifact. The
> execution copy + todo live in the gitignored `asima-parent/tasks/`.

## Overview

Add a **leave-balance/quota layer** on top of the existing `leave_requests`
module (which today has approval chains but *no* balances — see
`leave-requests.constants.ts`: "quotas, accrual, and balances are a v1
conversation"). This *is* that v1.

Requirements:

1. Every employee defaults to **10 sick + 10 vacation** days.
2. Admins **grant** sick / vacation / bereavement / birthday / emergency to any
   employee, any amount, **any number of times**.
3. Leave applies **against the employee's work schedule** — you cannot submit
   leave on dates that don't coincide with your schedule.
4. `/employee/leaves` revamp: **balances summary** (available / used / pending),
   the requests **table**, and a **right-side drawer** to apply with a **live
   working-day count** (same day = 1; 1/20→1/21 = 2 workdays).

## Locked Decisions

| # | Decision | Choice |
|---|----------|--------|
| D1 | Balance consumption | **Reserve on submit.** pending reserves; approval finalizes; reject/cancel releases. `available = allowance − used(approved) − reserved(pending)`. |
| D2 | Day counting | **Work-schedule-aware.** Chargeable days = scheduled working days in the inclusive range. Boundary/past-date rules in D8. |
| D3 | Leave types | **Replace enum** → `vacation, sick, bereavement, birthday, emergency`. |
| D4 | Grants | **Append-only ledger** (`leave_allocations`). `allowance = SUM(amount)` per (employee, type). |
| D5 | Migrations | **Fresh project — edit existing migration files in place.** No alter/backfill migrations. One migration per table. |
| D6 | Default grants | Created **on user creation** (service) **and** through the existing **user seeder** — no new overlapping seeder file. |
| D7 | Grant reversibility (C1) | **Grants only add** — `CHECK (amount > 0)`, no negative amounts. If revocation is ever needed (v1+), soft-delete the specific allocation row (`deleted_at`); excluded from the `allowance` sum. Keeps `available` non-negative. |
| D8 | Submit date rules | **No past-dated leave** (`start_date >= today`), and **both `start_date` and `end_date` must be scheduled workdays** for the applying employee. Non-workday boundary or past date → 422. |

## Migration & Seeder Strategy (D5 / D6) — read first

This is **not deployed**. There is **no existing leave data** (no leave-request
or allocation seeder exists). So:

- **No `Alter…` or backfill migrations.** Edit the existing
  `1778400000000-CreateLeaveRequestsTable.ts` *in place* to: (a) define the new
  enum values directly, and (b) add the `working_days` column as `NOT NULL`
  from creation. Nothing to remap because nothing is deployed.
- **Exactly one new migration:** `CreateLeaveAllocationsTable` (a genuinely new
  table — one migration per table keeps files uniform/organized).
- **`db:fresh` is the reset path** — `schema:drop && migration:run && seed`
  rebuilds cleanly every time.
- **Seeder:** extend the existing `UserSeedService` so that after each user is
  inserted it also inserts that user's **default allocations** (10 sick, 10
  vacation), idempotent by natural key `(employee_id, leave_type,
  source='default')`. No separate/overlapping seed file. Both the seeder and
  `UsersService.create` share one `DEFAULT_LEAVE_ALLOCATIONS` constant + the
  allocation repository so the "every employee starts at 10/10" rule has a
  single definition.

## Architecture Decisions

- **`allowance` is summed from the ledger, never stored as a running total.**
  Default 10/10 are two `source='default'` rows; admin grants append more.
- **`working_days` is snapshotted onto `leave_requests` at submit** (NOT NULL
  column in the create migration), like `l1_approver_id` already is. Makes
  `reserved`/`used` cheap aggregates, immune to later schedule edits.
- **Balance is computed in a service, not materialized.** No `leave_balances`
  table — `LeaveBalanceService` derives `{allowance, used, reserved,
  available}` per type from `leave_allocations` + `leave_requests`.
- **Balance is enumerated over the leave-type ENUM, not the ledger rows (C2).**
  The service iterates all 5 `LEAVE_TYPES` and defaults `allowance/used/
  reserved` to 0 for any type with no rows. Computing via `GROUP BY leave_type`
  over `leave_allocations` alone would **drop** bereavement/birthday/emergency
  (which start with zero grants) from the response — the summary would silently
  show only sick + vacation. Always return exactly 5 rows. (See Concurrency &
  Balance Computation below for the query shape.)
- **Day counting + date rules live in one `LeaveDayCountService`** — given
  `(employee_id, start, end)`, resolves each date's weekday against the
  `work_schedule` effective on that date; exposes `isWorkday(emp, date)` and
  `countWorkingDays(...)`. Submit and the preview endpoint both run the same D8
  checks (no past dates; start & end must be workdays) through this one service,
  so the drawer's inline validation can never diverge from server enforcement.
- **Reserve-on-submit is serialized by a row lock (C3).** See the dedicated
  section below — submit runs in a transaction that `SELECT … FOR UPDATE`s the
  employee's allocation rows for the requested type *before* computing
  `available`, so concurrent submits of the same type can't both pass the check.

## Concurrency & Balance Computation

**Balance computation (C2) — 2 grouped queries + enum merge, never per-type loop:**

```
allocAgg  = SELECT leave_type, SUM(amount)         FROM leave_allocations
            WHERE employee_id = $1 AND deleted_at IS NULL GROUP BY leave_type
reqAgg    = SELECT leave_type, status, SUM(working_days) FROM leave_requests
            WHERE employee_id = $1 AND deleted_at IS NULL
              AND status IN ('approved','pending_l1','pending_l2')
            GROUP BY leave_type, status
// then, for EACH of the 5 LEAVE_TYPES (source of truth = the enum):
//   allowance = allocAgg[type] ?? 0
//   used      = reqAgg[type]['approved'] ?? 0
//   reserved  = reqAgg[type][pending_*] ?? 0
//   available = max(0, allowance − used − reserved)
```

Two round-trips total regardless of type count. Existing indexes
`leave_requests(employee_id, status)` and `leave_allocations(employee_id,
leave_type)` cover both filters.

**Reserve-on-submit serialization (C3) — `SELECT … FOR UPDATE` on the
allocation rows:** submit runs inside one repository transaction:

```
BEGIN
1. SELECT amount FROM leave_allocations
     WHERE employee_id = $emp AND leave_type = $type AND deleted_at IS NULL
     FOR UPDATE;                       -- allowance, AND the lock
2. compute used + reserved for ($emp, $type) from leave_requests
3. if (allowance − used − reserved) < working_days  →  422, ROLLBACK
4. INSERT the pending leave_request (status = pending_l1, working_days snapshot)
COMMIT
```

Why this is correct: the allocation rows are the **shared resource both
concurrent submits contend on**. Submit B blocks at step 1 until submit A
commits, then re-reads `reserved` — which now includes A's just-inserted
pending row — and correctly rejects if it would overspend. A type with **zero**
allocations has `available = 0`, so any submit fails the step-3 check regardless
of races (no double-spend possible even though there are no rows to lock). The
leave-allocation repo exposes `sumForUpdate(manager, emp, type)`; the request
insert joins the **same** `EntityManager`/transaction.

## Data Model

```
leave_allocations  (NEW table — one new migration)
  id, employee_id FK users RESTRICT, leave_type (new enum),
  amount int CHECK (amount > 0),   -- C1: grants only ADD; revoke = soft-delete a row, never a negative amount
  source enum('default','admin_grant'),
  reason varchar(500) null, granted_by FK users null,
  created_by/updated_by/deleted_by, created_at/updated_at/deleted_at
  INDEX (employee_id, leave_type)

leave_requests  (EXISTING migration edited in place)
  leave_type enum  →  vacation, sick, bereavement, birthday, emergency
  + working_days   int NOT NULL CHECK (working_days >= 1)
```

Derived per (employee, type) in `LeaveBalanceService`:

```
allowance = SUM(leave_allocations.amount)        (not deleted)
used      = SUM(leave_requests.working_days)     status = 'approved'
reserved  = SUM(leave_requests.working_days)     status ∈ {pending_l1, pending_l2}
available = allowance − used − reserved
```

## Dependency Graph

```
edit CreateLeaveRequestsTable (enum + working_days, D5)
        │
        ├─ LeaveDayCountService ─ submit enforcement (D2)
        │
CreateLeaveAllocationsTable ─ default grants (svc + seeder, D6) ─ LeaveBalanceService
        │                                                              │
        └─────────────────── admin grant endpoints ───────────────────┤
                                                                       ▼
                FE api/schemas ─ balances UI ─ apply drawer ─ requests table
```

---

## Task List

### Phase 0 — Model foundations

#### Task 1: Replace `leave_type` enum (edit existing migration in place)
**Description:** Change the enum to `vacation, sick, bereavement, birthday,
emergency` by editing `1778400000000-CreateLeaveRequestsTable.ts` directly
(the `CREATE TYPE "leave_type"` line) plus backend constants/entity/domain/DTOs
and FE schemas/labels. No new migration, no remap (fresh project).

**Acceptance criteria:**
- [ ] `CreateLeaveRequestsTable` migration `CREATE TYPE` lists the 5 new values; `down()` unchanged-compatible.
- [ ] `LEAVE_TYPES`, entity enum, domain enum, submit DTO, FE `LEAVE_TYPES`/`LEAVE_TYPE_LABELS` list exactly the 5 types; FE default `annual → vacation`.
- [ ] `db:fresh` runs clean; grep shows no `annual`/`unpaid`/`other` remaining.

**Verification:** `npm run migration:run` from clean schema succeeds; backend `build` + FE `typecheck` pass.
**Dependencies:** None
**Files:** `migrations/1778400000000-CreateLeaveRequestsTable.ts`, `leave-requests.constants.ts`, `entities/leave-request.entity.ts`, `domain/leave-request.ts`, `dto/me/submit-leave-request.dto.ts`, `dto/admin/*`, FE `features/leave/schemas.ts`, `features/leave/format.ts`
**Scope:** M

#### Task 2: Day-count service + submit date rules + `working_days` (edit existing migration in place)
**Description:** Add `working_days NOT NULL CHECK (>=1)` to the **existing**
`CreateLeaveRequestsTable` migration (not an alter). Add
`LeaveDayCountService.countWorkingDays(employee_id, start, end)` resolving each
date against the `work_schedule` effective that date. `submit` computes
`working_days`, applies the **date rules (D8)** below, and stores the snapshot.

**Date rules (D8) — enforced in submit AND the day-count preview (Task 7):**
1. **No past dates** — `start_date >= today` (today = current date in the
   configured business timezone, date-only). Since `end_date >= start_date`,
   the end is covered too.
2. **`start_date` must be a scheduled workday** for this employee (a
   `work_schedule` row exists for that date's weekday, effective on that date).
3. **`end_date` must be a scheduled workday** for this employee (same rule).
4. `working_days` = scheduled workdays in `[start, end]` inclusive. With rules
   2–3, this is always ≥ 1, so the old "reject if 0 workdays" is now *subsumed*
   — a non-workday boundary is rejected first with a clearer message.

Each rule returns a distinct 422 field so the drawer can show it inline
(`{start_date: "Leave can't start in the past."}`, `{start_date: "Not a working
day on your schedule."}`, `{end_date: "..."}`).

**Acceptance criteria:**
- [ ] `countWorkingDays` correct for: single workday (1), Fri→Mon spanning weekend (Fri+Mon = 2), mid-range schedule change (per-date row).
- [ ] Submit **rejects**: `start_date` < today (past); `start_date` on a non-workday; `end_date` on a non-workday — each with its own 422 field/message.
- [ ] Submit **accepts** a same-day workday request (working_days = 1) and a Fri→Mon request (working_days = 2); stores the correct `working_days` snapshot.
- [ ] `working_days` column present in the create migration; entity + domain carry it.
- [ ] Employee with **no work schedule at all** → every date is a non-workday → submission blocked (rule 2).

**Verification:** unit spec for the service (count cases) + `is-workday` helper; `leave-requests.service.spec.ts` covers past-date, non-workday-start, non-workday-end blocks + valid same-day/Fri→Mon snapshots.
**Dependencies:** Task 1
**Files:** `leave-requests/leave-day-count.service.ts` (+spec), `migrations/1778400000000-CreateLeaveRequestsTable.ts`, `entities/leave-request.entity.ts`, `domain/leave-request.ts`, `leave-requests.service.ts`, `leave-requests.module.ts`
**Scope:** M

#### Checkpoint: Phase 0
- [ ] `db:fresh` clean; leave specs green. Human review of enum + the D8 date rules (no past dates, workday boundaries).

### Phase 1 — Balances backend

#### Task 3: `leave_allocations` ledger (one new migration + hexagonal slice)
**Description:** New table via **one** new migration `CreateLeaveAllocationsTable`;
full slice: entity, mapper, repository (port + concrete), domain class,
`CreateAllocationInput`. Append-only (no update path).

**Acceptance criteria:**
- [ ] `CreateLeaveAllocationsTable` migration: FKs, `amount > 0` CHECK, audit + soft-delete cols, `(employee_id, leave_type)` index.
- [ ] Repo can `create`, `sumByEmployeeAndType`, `listForEmployee`.

**Verification:** `migration:run` from clean schema; mapper spec round-trips.
**Dependencies:** Task 1
**Files:** `leave-allocations/` (domain/persistence/module), `migrations/<ts>-CreateLeaveAllocationsTable.ts`
**Scope:** L

#### Task 4: Default 10/10 on user create + via existing user seeder
**Description:** Shared `DEFAULT_LEAVE_ALLOCATIONS` constant (sick 10, vacation
10). `UsersService.create` inserts the two `source='default'` rows. Extend
`UserSeedService` to insert the same rows per user (idempotent by
`employee_id+leave_type+source`). **No new seeder file.**

**Acceptance criteria:**
- [ ] Admin-created user → exactly 2 default allocations.
- [ ] `npm run seed` twice = no-op second run; every seeded active employee has the 2 rows.

**Verification:** `users.service.spec.ts` asserts defaults; seed idempotency check.
**Dependencies:** Task 3
**Files:** `users/users.service.ts`, `leave-allocations/leave-allocations.constants.ts`, `database/seeds/user/user-seed.service.ts`, `seed.module.ts`
**Scope:** M

#### Task 5: Balance service + `GET /users/me/leave-balances` + reserve-on-submit
**Description:** `LeaveBalanceService.forEmployee(id)` → per-type
`{allowance, used, reserved, available}`. Self-service endpoint exposes it.
`submit` gains the transactional check `available(type) ≥ working_days` else 422.

**Acceptance criteria:**
- [ ] **(C2)** `GET /users/me/leave-balances` returns **exactly 5 rows — one per `LEAVE_TYPES` value** — even for types with no allocation rows (bereavement/birthday/emergency → `allowance:0, used:0, reserved:0, available:0`). Service iterates the enum, not the ledger rows. Computed via the 2 grouped queries in "Concurrency & Balance Computation" (no per-type query loop).
- [ ] Over-`available` submit → 422 `{balance:"..."}`; pending reserves; cancel/reject releases (next read reflects it).
- [ ] **(C3)** Submit runs in one transaction that `SELECT … FOR UPDATE`s the `(employee_id, leave_type)` allocation rows **before** computing `available`, then inserts. Repo exposes `sumForUpdate(manager, emp, type)`; the request insert uses the **same** `EntityManager`.
- [ ] **(I1)** Approve/reject/auto-approve paths add **no** balance-write code — status change alone moves a row reserved→used (balance is computed). Confirm no redundant mutation is introduced.

**Verification:** service spec (reserve→finalize→release moves the numbers; zero-allocation type returns 0s, not absent); **concurrency e2e**: two parallel submits that each fit alone but not together → exactly one 201 + one 422; e2e over-limit 422 + cancel-restores.
**Dependencies:** Task 2, Task 3
**Files:** `leave-requests/leave-balance.service.ts` (+spec), `controllers/me-leave-requests.controller.ts`, `leave-requests.service.ts`, `leave-allocations` repo (`sumForUpdate`), `domain/leave-balance.ts`
**Scope:** M

#### Task 6: Admin grant endpoints + permissions
**Description:** `POST /admin/users/:id/leave-allocations` (grant),
`GET /admin/users/:id/leave-allocations` (history), optional
`GET /admin/users/:id/leave-balances`. New `LEAVE_ALLOCATION` resource
(Create/View) added to `permissions.json` + `roles.json` (edited in place).

**Acceptance criteria:**
- [ ] `LEAVE_ALLOCATION:Create` holder grants any type/amount any number of times → balance grows; without → 403.
- [ ] `npm run seed` idempotently grants the new permission to HR_ADMIN/SUPER_ADMIN.

**Verification:** e2e grant → balance reflects; `seeds-grant-matrix.spec.ts` updated.
**Dependencies:** Task 3, Task 5
**Files:** `leave-allocations/controllers/admin-leave-allocations.controller.ts`, `dto/admin/grant-allocation.dto.ts`, `leave-allocations.service.ts`, `permissions.constants.ts`, `seeds/data/permissions.json`, `seeds/data/roles.json`
**Scope:** M

#### Checkpoint: Phase 1
- [ ] `test` + `test:e2e` green; `db:fresh` gives every employee 10 sick / 10 vacation available. Review balance API shape.

### Phase 2 — Frontend revamp (`/employee/leaves`)

#### Task 7: API client + schemas + day-count preview endpoint
**Description:** Extend FE `api.ts`/`schemas.ts` with `me.balances()`,
`me.dayCountPreview(start,end)`, admin `grant()`, new types. Add backend
`GET /users/me/leave-requests/day-count` returning `{working_days}` on success,
or a **422 carrying the same D8 field errors** submit would raise (past date,
non-workday start/end) — so the drawer validates exactly as submit enforces.

**Acceptance criteria:** typed methods + zod schemas; preview returns the same count **and the same D8 422s** submit would; a past date or non-workday boundary in the preview yields the field-specific error.
**Verification:** FE `typecheck`; mocked client test.
**Dependencies:** Task 5
**Files:** backend `me-leave-requests.controller.ts`, FE `features/leave/api.ts`, `features/leave/schemas.ts`
**Scope:** M

#### Task 8: Balances summary on the page
**Description:** Summary (cards/table) per type: **available / used / pending**.
**Acceptance criteria:** renders 5 types from `me.balances()`; loading/error states.
**Verification:** component test with mocked balances.
**Dependencies:** Task 7
**Files:** `features/leave/components/employee-leaves-page.tsx`, `features/leave/components/leave-balance-summary.tsx`
**Scope:** S

#### Task 9: "Apply for leave" right-side drawer + live day count
**Description:** Move New-request into a right-side **Sheet** (existing
`components/ui/sheet.tsx`) opened by "Apply for leave". On date change,
debounced preview shows **"This request = N working day(s)"** and surfaces the
schedule 422 inline; same-day → 1; range → working-day count.

**Acceptance criteria:** drawer from the right with type/start/end/reason + live count; submit closes + refreshes table & balances; no-workday range disables submit with the schedule message.
**Verification:** component test for count display + disabled-on-422.
**Dependencies:** Task 7
**Files:** `features/leave/components/apply-leave-drawer.tsx`, `employee-leaves-page.tsx`
**Scope:** M

#### Task 10: Requests table — Days column + counts wired to summary
**Description:** Add **Days** (`working_days`) column; ensure cancel refreshes
table + balances.
**Acceptance criteria:** table shows working-days; cancel updates both views.
**Verification:** `employee-leaves-page.spec.tsx` updated.
**Dependencies:** Task 8, Task 9
**Files:** `employee-leaves-page.tsx`, `tests/unit/features/leave/employee-leaves-page.spec.tsx`
**Scope:** S

#### Checkpoint: Phase 2
- [ ] FE tests + typecheck green; manual 1-day/2-day reserve→release run-through against the running backend.

### Phase 3 — Admin grant UI + docs

#### Task 11: Admin "Grant leave" UI
**Description:** Admin grants type/amount/reason to an employee + views history
+ balances. On the admin leave/users surface.
**Acceptance criteria:** grant → employee's available updates; gated by `LEAVE_ALLOCATION:Create`.
**Verification:** component test; manual grant.
**Dependencies:** Task 6, Task 7
**Files:** `features/leave/components/admin-leave-page.tsx`, `features/leave/components/grant-leave-drawer.tsx`
**Scope:** M

#### Task 12: ADR + doc cleanup
**Description:** ADR `0002-leave-balances-and-allocations.md` (reserve-on-submit
+ computed balance + ledger). Correct the stale "no balances in v0" comment in
`leave-requests.constants.ts`.
**Acceptance criteria:** ADR committed; stale comments fixed.
**Dependencies:** Phases 0–2
**Files:** `asima-backend/docs/adr/0002-*.md`, `leave-requests.constants.ts`, CLAUDE.md
**Scope:** S

#### Checkpoint: Complete
- [ ] All criteria met; `test`, `test:e2e`, FE tests, both builds green; `db:fresh` + manual run-through clean; ready for PR.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Concurrent submits double-spend a balance | Med | `SELECT … FOR UPDATE` on the `(emp, type)` allocation rows serializes same-type submits (C3); concurrency e2e proves one 201 + one 422. |
| Zero-grant types missing from balance response | Med | Enumerate the 5 `LEAVE_TYPES`, default to 0 — never group over ledger rows alone (C2). |
| Leave with no work schedule at all | Med | Every date is a non-workday → submission blocked by D8 rule 2; admin assigns a schedule first. |
| Day-count vs DST / mid-range schedule change | Low | Date-only arithmetic; per-date schedule resolution; explicit unit cases. |
| Editing the existing create migration | Low | Fresh project, no deployed data; `db:fresh` rebuilds; verify up-run from clean schema. |

## Resolved Decisions (previously open)

- **Past-dated leave (D8)** — **blocked.** `start_date >= today`.
- **Workday boundaries (D8)** — `start_date` and `end_date` must both be the
  employee's scheduled workdays.
- **Grant revocability (C1/D7)** — grants are **add-only** (`CHECK (amount > 0)`);
  no negative-amount corrections; revoke via soft-delete if ever needed.

## Open Questions (defaults chosen, non-blocking)

- **Birthday leave** cap — default: admin-granted only, no special cap.
- **Zero-balance types** (bereavement/birthday/emergency start at 0) — not
  selectable in the drawer until a grant exists (available 0 ⇒ blocked).
- **Reversing an *approved* leave** (S5) — out of v1 scope; cancel works only on
  pending. Revisit if users need to give approved `used` days back.
